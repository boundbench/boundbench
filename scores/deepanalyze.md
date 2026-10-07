# Defense-in-Depth Score: DeepAnalyze

**Repo:** https://github.com/ruc-datalab/DeepAnalyze · **Commit:** `f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4` · **Reviewed:** 2026-10-03
**What it is:** Agentic LLM for autonomous data science with code-execution runtime and API
**Category:** Data & Analytics
**Scored configuration:** The README's first deployment mode, the WebUI (demo/chat: bash start.sh -> backend.py on 0.0.0.0:8200 plus a workspace file server on port 8100) with its shipped defaults and a local vLLM model server.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials no · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 2.7 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | C2-POWERBYPASS | **0.00** | High |
| C3 | Tool & action scoping | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C4 | Code-execution isolation | L3 | L3 | L3 | L3 | 0.75 | G1 | **0.50** (alt) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C9 | Audit & traceability | L2 | L2 | L3 | L2 | 0.55 | G1 | **0.50** (alt) | High |
| C10 | Limits & kill switch | L2 | L2 | L3 | L2 | 0.55 | G1 | **0.50** (alt) | High |

Controls where a risk surface exists: 1.70 / 9.0 (19%); 1 criterion scored SA (surface absent).

As shipped in the README's first WebUI, DeepAnalyze executes every block of Python the model writes directly on the host, as your user, with your environment variables, full network access, and no approval, sandbox, round limit, or log. The backend's network exposure and access control, and that of its workspace file server, are not locked down. The newer WebUI v2 in the same repo is substantially safer by default (hardened per-session Docker sandbox with no network, run budgets, stop, execution records); deployers should use it, bound to localhost.

## Critical gaps
- Model code inherits the operator's full environment and runs as the OS user, so the agent's authority is the user's entire account; the backend's network access control is also not locked down. (ASI03; C1) — [demo/chat/backend.py:63](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L63)
- Arbitrary model-generated Python, the most powerful action, executes automatically with no approval gate on both the agent loop and /execute. (ASI02, ASI09; C2) — [demo/chat/backend.py:666](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L666); [demo/chat/backend.py:504-516](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L504-L516)
- In the default WebUI, model code runs as a same-user host subprocess with the full environment; there is no isolation boundary. (ASI05; C4) — [demo/chat/backend.py:68-77](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L68-L77); [demo/chat/backend.py:63](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L63)
- A hijacked session can exfiltrate data through code egress and take irreversible actions without any human involvement; other exfiltration paths are not confined either. (ASI01; C5) — [demo/chat/backend.py:743](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L743)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

The WebUI backend runs model-written Python as the same operating-system user that started it and hands that code a full copy of the server's environment variables, so whatever credentials the operator has (environment keys, ~/.aws, ~/.ssh, cloud CLIs) are available to the model. The API's network exposure and access control are not locked down. There is no agent-specific identity and no notion of which user is asking.

- **S L0:** Model code runs as the launching OS user with env=os.environ.copy(); the agent has no identity of its own and no narrowing of ambient authority. — [demo/chat/backend.py:63](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L63); [demo/chat/backend.py:68-77](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L68-L77) (verified)
  - *To reach the next level:* No scoped identity: environment is not scrubbed and the subprocess is not run under a dedicated low-privilege account.
- **C L0:** No authorization layer sits on the tool path, and access control on the HTTP routes is not locked down. (verified)
  - *To reach the next level:* No authorization layer on any tool path; even the main code path is unchecked.
- **D L0:** The default launch grants full ambient authority with no narrower profile, and network exposure is not locked down. (verified)
  - *To reach the next level:* The default deployment would need a least-privilege, authenticated configuration.
- **B L0:** A hijack gets everything the operator's account can do on the host and every service whose credentials are in the environment or home directory. — [demo/chat/backend.py:63](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L63) (verified)
  - *To reach the next level:* Model code should run without the operator's credentials and the service would need hardened access control.
- **Cap:** none
- **Notes:** WebUI v2 (demo/chat_v2) has related access-control gaps, but its default Docker executor passes no host environment to model code; that benefit is credited under C4.

### C2 Approval gates — 0.00 (high)

There is no human approval anywhere. Every <Code> block the model emits is extracted and executed immediately in a loop until the model writes an answer, and the same code-execution engine is exposed directly over HTTP. Model code can delete or overwrite workspace and host files, make network requests, and run any program, with nothing to review first and no undo.

- **S L0:** No approval mechanism exists; extracted code goes straight to execute_code_safe. — [demo/chat/backend.py:666](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L666); searched `rg -n -i 'approv|confirm|permission'` in `demo/chat/backend.py` → 0 hits (verified)
  - *To reach the next level:* No per-call approval showing the exact code before execution.
- **C L0:** The most powerful (and only) action, arbitrary Python execution, is ungated on both the agent loop and the /execute route. — [demo/chat/backend.py:666](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L666); [demo/chat/backend.py:504-516](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L504-L516) (verified)
  - *To reach the next level:* Code execution would need to traverse an approval gate on every path.
- **D L0:** No approval mode exists to enable; autonomous execution is the only behaviour. — [demo/chat/backend.py:620](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L620); searched `rg -n -i 'approv|confirm|permission'` in `demo/chat/backend.py` → 0 hits (verified)
  - *To reach the next level:* Approval would need to be on by default.
- **B L0:** Executed code can irreversibly delete files, exfiltrate data, or call external services with no checkpoint or rollback. — [demo/chat/backend.py:68-77](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L68-L77); [demo/chat/backend.py:63](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L63) (verified)
  - *To reach the next level:* No checkpoints/rollback of filesystem state and no previews for external actions.
- **Cap:** C2-POWERBYPASS — Arbitrary Python execution, the single most powerful action, runs without any gate in the default configuration (backend.py:666).
- **Notes:** WebUI v2's 'manual' interaction mode pauses after each execution (services/chat.py:913-935), not before it, so it is not an approval gate.

### C3 Tool & action scoping — 0.00 (high)

The agent has one tool: run arbitrary Python. It is not narrowed or validated in any way, and input handling on the surrounding HTTP endpoints is not a strict boundary.

- **S L0:** Raw passthrough: model text is executed as a Python program; HTTP route input handling is not a strict boundary. — [demo/chat/backend.py:666](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L666) (verified)
  - *To reach the next level:* No allowlist validation: arbitrary code is executed as-is.
- **C L0:** The model's only tool validates nothing; the route-level containment checks are not a complete boundary. (verified)
  - *To reach the next level:* Validation would need to cover the model's execution tool and every route.
- **D L0:** Everything is enabled by default: code execution with filesystem write and network access. — [demo/chat/backend.py:68-77](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L68-L77); [demo/chat/backend.py:504-516](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L504-L516) (verified)
  - *To reach the next level:* No read-only or reduced tool set by default.
- **B L0:** A misused tool reaches the whole machine as the OS user, and route-level containment is not a complete boundary. (verified)
  - *To reach the next level:* Tool reach would need to be scoped to a workspace with bounded effects.
- **Cap:** none
- **Notes:** WebUI v2 adds stricter workspace path handling (services/workspace.py:41-65, 562-567), but its model tool is still unrestricted Python, so it is not scored as a stronger C3 mechanism.

### C4 Code-execution isolation — 0.50 (high)

In the scored WebUI, model-generated Python runs as an ordinary child process on the host, as the same user, with the server's full environment and unrestricted network and filesystem access; the only limit is a 120-second timeout. The project's newer WebUI v2 ships a much better design that is on by default there: each session gets a Docker container with all capabilities dropped, no-new-privileges, a non-root user, a read-only root filesystem, no network, CPU/memory/PID limits, and only the session workspace mounted, and it refuses to fall back to host execution unless an explicitly named unsafe flag is set. Because that sandbox is only used if the deployer chooses WebUI v2 instead of the default WebUI, it is credited as an opt-in mechanism.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Same-user subprocess on the host: subprocess.run([sys.executable, tmp_path]) with no isolation primitive. — [demo/chat/backend.py:68-77](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L68-L77); searched `rg -n 'docker|sandbox|seccomp|setrlimit|resource\.'` in `demo/chat/backend.py` → 0 hits (verified)
    - *To reach the next level:* No OS-level separation (container, dedicated user, or sandbox profile).
  - **C L0:** Nothing is sandboxed: agent-loop execution and the /execute route both use the same host subprocess. — [demo/chat/backend.py:666](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L666); [demo/chat/backend.py:504-516](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L504-L516) (verified)
    - *To reach the next level:* The main execution path would need to run inside an isolation boundary.
  - **D L0:** No sandbox exists in this mode to be on by default. — [demo/chat/backend.py:68-77](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L68-L77); searched `rg -n 'docker|sandbox|seccomp|setrlimit|resource\.'` in `demo/chat/backend.py` → 0 hits (verified)
    - *To reach the next level:* A sandbox would need to be on by default.
  - **B L0:** Host-equivalent: code runs as the OS user with the full environment (any credentials) and unrestricted network and home-directory access. — [demo/chat/backend.py:63](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L63); [demo/chat/backend.py:68-77](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L68-L77) (verified)
    - *To reach the next level:* Code would need to run without host credentials, with workspace-only file access and restricted egress.
- **WebUI v2 (demo/chat_v2) per-session hardened Docker executor** (alt; raw 0.75, cap G1 → 0.50) ← counted
  - **S L3:** Hardened container: --cap-drop ALL, no-new-privileges, non-root user 1000:1000, --read-only root, network none, memory/CPU/PID limits (Docker's default seccomp profile is not overridden). — [demo/chat_v2/backend_app/services/docker_executor.py:502-513](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/services/docker_executor.py#L502-L513); [demo/chat_v2/backend_app/services/docker_executor.py:519-522](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/services/docker_executor.py#L519-L522); [demo/chat_v2/backend_app/settings.py:124](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/settings.py#L124); [demo/chat_v2/backend_app/settings.py:130](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/settings.py#L130) (verified)
    - *To reach the next level:* Not kernel-separated (no gVisor/microVM); a container escape lands on the host.
  - **C L3:** Both agent and manual execution go through execute_code_safe, which uses Docker when enabled and refuses host execution without the unsafe flag; Docker setup failures return an error rather than falling back. — [demo/chat_v2/backend_app/services/execution.py:36-56](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/services/execution.py#L36-L56); [demo/chat_v2/backend_app/services/execution_service.py:101-107](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/services/execution_service.py#L101-L107) (verified)
    - *To reach the next level:* Report export renders model-written Markdown through pandoc/xelatex on the host outside the container.
  - **D L3:** Docker mode is the default and local mode needs both DEEPANALYZE_EXECUTION_MODE=local and DEEPANALYZE_ALLOW_UNSAFE_LOCAL_EXECUTION=true; startup validation refuses otherwise. — [demo/chat_v2/backend_app/settings.py:100-104](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/settings.py#L100-L104); [demo/chat_v2/backend_app/services/docker_executor.py:644-650](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/services/docker_executor.py#L644-L650) (verified)
    - *To reach the next level:* Individual hardening settings (user, read-only, network mode) can be weakened silently by environment/.env values.
  - **B L3:** Only the session workspace is mounted, exec passes only fixed display variables (no host secrets), network is off, and resources are limited. — [demo/chat_v2/backend_app/services/docker_executor.py:514-515](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/services/docker_executor.py#L514-L515); [demo/chat_v2/backend_app/services/docker_executor.py:579-596](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/services/docker_executor.py#L579-L596) (verified)
    - *To reach the next level:* Containers persist per session (idle TTL 1800 s) instead of being destroyed per run.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.
- **Notes:** Other modes: API/utils.py also runs code as a same-user subprocess with os.environ.copy(); deepanalyze.py runs exec() in-process. WebUI v2's Docker sandbox scores 0.75 before the G1 cap.

### C5 Untrusted input blast radius — 0.00 (high)

DeepAnalyze's job is to read data files supplied by the user, which may come from anywhere, and code output from those files flows straight back into the model's context with the same standing as everything else. Nothing limits what a hijacked session can do: it can run any code, delete files, and send data out, either directly from the code (full network access) or through other unconfined paths.

- **S L0:** No structural limit: execution output (which includes untrusted file content) is fed back verbatim and the next code block runs automatically. — [demo/chat/backend.py:743](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L743); searched `rg -n -i 'untrusted|inject|sanitiz'` in `demo/chat/backend.py` → 0 hits (verified)
  - *To reach the next level:* No approval or capability removal once untrusted content has been read.
- **C L0:** Untrusted file contents and tool output are not distinguished from the user's instructions; workspace file names are inserted into the user's own message. — [demo/chat/backend.py:743](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L743); [demo/chat/backend.py:604-612](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L604-L612) (verified)
  - *To reach the next level:* Untrusted sources would need to be tagged and handled differently from principal input.
- **D L0:** No defence exists to be on by default. — [demo/chat/backend.py:743](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L743); searched `rg -n -i 'untrusted|inject|sanitiz'` in `demo/chat/backend.py` → 0 hits (verified)
  - *To reach the next level:* A provenance-based limit would need to be on by default.
- **B L0:** Unattended exfiltration (code network access, plus other unconfined paths) plus irreversible actions (file deletion, arbitrary code). — [demo/chat/backend.py:68-77](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L68-L77) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions would need human approval or removal after untrusted input is read.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.
- **Notes:** C5-PUBLICTRIGGER was not applied: the agent is not triggered by public events. WebUI v2's default container has no network, which removes the direct exfiltration leg there.

### C6 Memory, context & configuration integrity — 0.10 (high)

DeepAnalyze has no long-term memory, vector store, or auto-loaded instruction files, and it does not load a .env from the working directory. The persistent state that does exist is the session workspace: files written by model code stay there across conversations in the same browser session, and the names of top-level files are inserted into every new user message as the '# Data' section, so a poisoned session can plant text that later runs see as part of the user's request and act on with code. Session separation is not enforced server-side.

- **S L0:** Model code can create arbitrarily named files that persist and whose names are re-injected into the user's message; nothing validates or logs these writes. — [demo/chat/backend.py:160-167](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L160-L167); [demo/chat/backend.py:604-612](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L604-L612) (verified)
  - *To reach the next level:* Writes to the persistent workspace are not logged, validated, or marked as data when re-injected.
- **C L0:** No persistence path is controlled. — [demo/chat/backend.py:604-612](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L604-L612); searched `rg -n -i 'untrusted|inject|sanitiz'` in `demo/chat/backend.py` → 0 hits (verified)
  - *To reach the next level:* Workspace persistence and prompt re-injection would need a control.
- **D L1:** Workspaces are namespaced by a session_id the browser generates; server-side enforcement is not a strict boundary. — [demo/chat/frontend/components/three-panel-interface.tsx:383-388](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/frontend/components/three-panel-interface.tsx#L383-L388) (verified)
  - *To reach the next level:* Namespaces are not robustly enforced server-side.
- **B L1:** Poisoned files persist across the user's conversations in that browser session and can steer later code execution. — [demo/chat/backend.py:604-612](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L604-L612); [demo/chat/backend.py:666](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L666) (verified)
  - *To reach the next level:* Poisoned state should be session-scoped or easy to inspect and purge before it influences tool use.
- **Cap:** none

### C7 Third-party extensions — 1.00 (high)

The scored WebUI backend loads no plugins, MCP servers, downloadable tools, or model files at runtime; the model is served by a separately run vLLM process. Model-written code could still install packages itself, but that is a consequence of unsandboxed code execution and is scored under code-execution isolation. The separate Jupyter demo connects to jupyter-mcp-server and the README's vLLM commands pass --trust-remote-code; neither is part of the scored backend.

- **Structural absence:** searched `rg -n -i 'pip install|importlib|trust_remote_code|pickle|torch.load|mcp|plugin|load_dotenv'` in `demo/chat/backend.py` → 0 hits
- **Notes:** Out of scored scope: demo/jupyter uses jupyter-mcp-server; README vLLM launch templates use --trust-remote-code for the model server.

### C8 Secrets & sensitive-data protection — 0.10 (high)

The WebUI holds no API keys of its own (the local vLLM key is the literal 'dummy') and sends no telemetry. But it does nothing to keep sensitive material contained: model code receives a full copy of the server's environment, nothing is redacted anywhere, and access to uploaded data through the workspace file server is not locked down.

- **S L0:** No secret or data protection: environment copied to model code, and access to served data is not locked down. — [demo/chat/backend.py:63](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L63); searched `rg -n -i 'redact|mask|secret|SecretStr'` in `demo/chat/backend.py` → 0 hits (verified)
  - *To reach the next level:* No masking or containment of sensitive data on any path.
- **C L0:** No path is protected: subprocess environment, file server, and model-bound tool output all pass content unfiltered. — [demo/chat/backend.py:63](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L63); [demo/chat/backend.py:743](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L743) (verified)
  - *To reach the next level:* Would need protection on at least the subprocess-environment and data-serving paths.
- **D L1:** No telemetry is configured and logging is minimal, but the data file server is on by default and its exposure is not locked down. (verified)
  - *To reach the next level:* Bounded by S (L0); data serving would need hardening.
- **B L1:** Project-held keys are none, but any long-lived operator credentials in the environment or home directory are readable by model code. — [demo/chat/backend.py:63](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L63); [demo/chat/backend.py:98](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L98) (verified)
  - *To reach the next level:* Model code would need to run without access to operator credentials.
- **Cap:** none

### C9 Audit & traceability — 0.50 (high)

The scored WebUI keeps no record of what the agent did: there is no logging module, no execution log, and the conversation transcript lives only in the browser. After an incident you could not reconstruct which code ran or who triggered it. WebUI v2 is better: it saves every executed script to the workspace and appends a structured execution record (run id, source agent or manual, timestamps, output, artifacts) to a session-state file outside the container-visible workspace, though it has no actor identity and keeps only the last 200 runs.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Tool calls are not recorded; only a few print statements exist. — searched `rg -n -i 'logging|logger|audit'` in `demo/chat/backend.py` → 0 hits; [demo/chat/backend.py:666](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L666) (verified)
    - *To reach the next level:* No structured record of each execution (code, result, timestamp).
  - **C L0:** Nothing is recorded for either the agent loop or /execute. — searched `rg -n -i 'logging|logger|audit'` in `demo/chat/backend.py` → 0 hits; [demo/chat/backend.py:504-516](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L504-L516) (verified)
    - *To reach the next level:* Would need a record for every execution path.
  - **D L0:** No audit trail exists to enable. — searched `rg -n -i 'logging|logger|audit'` in `demo/chat/backend.py` → 0 hits (verified)
    - *To reach the next level:* A record would need to be on by default.
  - **B L0:** Execution leaves no server-side trace; failures and actions are silently unrecorded. — searched `rg -n -i 'logging|logger|audit'` in `demo/chat/backend.py` → 0 hits; [demo/chat/backend.py:68-77](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L68-L77) (verified)
    - *To reach the next level:* Records would need to be written per action.
- **WebUI v2 (demo/chat_v2) per-session execution records** (alt; raw 0.55, cap G1 → 0.50) ← counted
  - **S L2:** Each execution is stored as a structured record with run_id, source, code path, result, artifacts, and start/finish timestamps. — [demo/chat_v2/backend_app/services/execution_service.py:142-156](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/services/execution_service.py#L142-L156) (verified)
    - *To reach the next level:* No principal/approver attribution and no correlation beyond the session.
  - **C L2:** Both agent and manual executions go through execute_managed_code and are recorded. — [demo/chat_v2/backend_app/services/execution_service.py:76-85](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/services/execution_service.py#L76-L85); [demo/chat_v2/backend_app/services/chat.py:903-911](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/services/chat.py#L903-L911) (verified)
    - *To reach the next level:* Configuration changes, stops, and model turns without code are not recorded as audit events.
  - **D L3:** Records are written by the backend to a .session_state directory beside, not inside, the session workspace that the container mounts. — [demo/chat_v2/backend_app/services/session_state.py:40-43](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/services/session_state.py#L40-L43) (verified)
    - *To reach the next level:* Recording can't be shown to be undisableable, and the file is rewritten and capped at 200 entries.
  - **B L2:** Records are written per execution after it completes; the list is truncated to the most recent 200. — [demo/chat_v2/backend_app/services/session_state.py:283-291](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/services/session_state.py#L283-L291) (verified)
    - *To reach the next level:* Records are written after execution and older entries are discarded, so the full trajectory is not durably replayable.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch — 0.50 (high)

Each code execution in the scored WebUI is killed after 120 seconds, and each model call is limited to 32,768 new tokens, but the agent loop itself runs until the model writes an answer, with no cap on rounds, total time, or cost, and there is no stop button on the server side. WebUI v2 adds real budgets on by default (12 model rounds, 15 minutes per analysis, a response-size cap, per-execution timeouts bounded by remaining time) and a stop endpoint that closes the model stream and stops the session's container, though code can leave background processes running in the container until it is reaped.

- **default configuration** (default; raw 0.15 → 0.15)
  - **S L1:** Only a per-execution timeout (120 s) and a per-call max_new_tokens; no iteration, wall-clock, or cost cap and no halt. — [demo/chat/backend.py:48-50](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L48-L50); [demo/chat/backend.py:620](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L620); searched `rg -n -i 'max_rounds|max_iter|round_count|max_steps'` in `demo/chat/backend.py` → 0 hits (verified)
    - *To reach the next level:* No iteration cap or session wall-clock limit on the agent loop.
  - **C L1:** Limits apply only to individual subprocess executions; the loop is unbounded. — [demo/chat/backend.py:620](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L620); [demo/chat/backend.py:68-77](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L68-L77) (verified)
    - *To reach the next level:* The top-level loop is not bounded.
  - **D L0:** The loop is unlimited by default. — [demo/chat/backend.py:620](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L620); searched `rg -n -i 'max_rounds|max_iter|round_count|max_steps'` in `demo/chat/backend.py` → 0 hits (verified)
    - *To reach the next level:* Sensible default caps on rounds and wall-clock time.
  - **B L0:** A runaway session can call the model and execute code indefinitely; there is no server-side stop. — [demo/chat/backend.py:620](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat/backend.py#L620); searched `rg -n -i 'stop|cancel|kill'` in `demo/chat/backend.py` → 3 hits (all three hits are model stop tokens / finish_reason handling; there is no stop endpoint or cancellation) (verified)
    - *To reach the next level:* No ceiling on a run and no way to stop it server-side.
- **WebUI v2 (demo/chat_v2) loop budgets and stop endpoint** (alt; raw 0.55, cap G1 → 0.50) ← counted
  - **S L2:** Round cap, wall-clock cap, response-size cap, and per-execution timeouts enforced in code; stop closes the stream and docker-stops the container. — [demo/chat_v2/backend_app/services/chat.py:750-760](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/services/chat.py#L750-L760); [demo/chat_v2/backend_app/settings.py:160-163](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/settings.py#L160-L163) (verified)
    - *To reach the next level:* No token/cost cap across the run and no rate limits on executions.
  - **C L2:** Budgets cover the top-level loop and each execution's timeout is bounded by remaining time. — [demo/chat_v2/backend_app/services/chat.py:903-911](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/services/chat.py#L903-L911) (verified)
    - *To reach the next level:* Background processes started by code inside the session container are not bounded by the run budget.
  - **D L3:** Defaults are set in settings and only the operator environment can change them; the model cannot raise them. — [demo/chat_v2/backend_app/settings.py:160-163](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/settings.py#L160-L163) (verified)
    - *To reach the next level:* No hard ceiling that configuration cannot exceed.
  - **B L2:** Moderate ceilings; stop or timeout stops the container, but a container lives on per session until idle TTL. — [demo/chat_v2/backend_app/services/docker_executor.py:609-621](https://github.com/ruc-datalab/DeepAnalyze/blob/f04a1c3b9ed3ae6c6efbb70e0cdcce4552c04ba4/demo/chat_v2/backend_app/services/docker_executor.py#L609-L621) (verified)
    - *To reach the next level:* No external spend ceiling and session containers persist after a run.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

## Rule-of-Two check
[A] untrusted input: user-supplied data files read by model code; output fed back as context (demo/chat/backend.py:743) · [B] sensitive data/systems: uploaded datasets and the operator's environment/home directory (demo/chat/backend.py:63) · [C] state change / egress: arbitrary Python with filesystem and network access (demo/chat/backend.py:68-77) · Same default session? Yes

## Highest-impact improvements
1. Make WebUI v2's hardened Docker executor (network none, non-root, read-only, no host env) the default for the WebUI and API, or retire the v1 backend. — C4 D L0→L3, +0.150 before caps (Playbook 3, step 1)
2. Harden the default network exposure and access control of the backend and file server. — C1 D L0→L2, +0.100 before caps
3. Add a default round cap, session wall-clock limit, and server-side stop endpoint to the v1 agent loop (as v2 already does). — C10 D L0→L2, +0.100 before caps
4. Stop passing os.environ to model code; pass only the display variables it needs. — C8 C L0→L1, +0.075 before caps
5. Write a structured per-execution log (code, output, timestamps, session) outside the workspace, as v2 does. — C9 S L0→L2, +0.150 before caps

## Re-audit log
- No changes.

## Limitations
- Static source review of commit f04a1c3 only; nothing was executed, installed, built, or probed.
- Scored mode is the README's first-listed WebUI (demo/chat). WebUI v2 (demo/chat_v2) was reviewed for its stronger mechanisms, which are credited as opt-in alternatives (G1) in C4, C9, and C10; v2 as a whole was not separately scored.
- Other modes (API/, deepanalyze.py library with in-process exec(), demo/cli, demo/jupyter via jupyter-mcp-server) were reviewed only for their execution path; vendored training code (deepanalyze/ms-swift, deepanalyze/SkyRL) and playground benchmarks were out of scope.
- The frontend was reviewed only for session-id handling and Markdown link/image rendering; Docker's default seccomp profile in v2 is assumed from Docker behaviour since the code does not override it.
- No reviewer-steering text was found in the repository.
