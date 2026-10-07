# Defense-in-Depth Score: XAgent

**Repo:** https://github.com/openbmb/xagent · **Commit:** `0dea79924347fad46e114621d7334d1136b30691` · **Reviewed:** 2026-10-04
**What it is:** Autonomous LLM agent from OpenBMB that plans and executes complex tasks with a dockerised ToolServer (shell, Python notebook, file editor, web browser).
**Category:** AI Assistants
**Scored configuration:** docker compose up with shipped docker-compose.yml, manager.yml and node.yml, then python run.py --task ... --config-file assets/config.yml (default --mode auto); XAgentServer web defaults footnoted.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents yes · external communication opt-in

## Score: 1.4 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | C2-POWERBYPASS | **0.00** | High |
| C3 | Tool & action scoping | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C4 | Code-execution isolation | L1 | L2 | L2 | L0 | 0.33 | C4-HOSTROOT | **0.25** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | C7-RCELOAD | **0.00** | High |
| C8 | Secrets & sensitive-data protection | L0 | L1 | L1 | L0 | 0.12 | — | **0.12** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L2 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |


XAgent runs a fully autonomous loop with a root shell, Python execution and unrestricted web access, and no human approval at any step. Its 'safe' ToolServer sandbox is a privileged Docker container that shares a network with the databases and mounts the shared config directory read-write, and the default deployment's credentials and access control are not locked down. A single prompt injection from a web page can therefore lead to host takeover and persistent changes to every later session.

## Critical gaps
- Agent tools run as root in a privileged container on the same network as the databases, giving a hijacked agent host-wide reach. (ASI03, T3; C1) — [assets/config/manager.yml:16](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L16); [ToolServer/ToolServerNode/core/tools/shell.py:27](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/tools/shell.py#L27)
- The root shell, the most powerful action, runs with no approval gate in the default auto mode. (ASI02, ASI09; C2) — [ToolServer/ToolServerNode/core/tools/shell.py:27](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/tools/shell.py#L27); [ToolServer/ToolServerNode/main.py:260](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/main.py#L260); [run.py:22](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/run.py#L22)
- The default code-execution sandbox is a privileged container running as root, which is host-equivalent. (ASI05, T11; C4) — [assets/config/manager.yml:16](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L16); [ToolServer/ToolServerNode/core/tools/shell.py:27](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/tools/shell.py#L27)
- Injected web content can drive an unattended root shell with unrestricted egress: exfiltration plus irreversible actions with no human in the loop. (ASI01, LLM01, T6; C5) — [XAgent/function_handler.py:244](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgent/function_handler.py#L244); [ToolServer/ToolServerNode/core/envs/web.py:104-106](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/envs/web.py#L104-L106); [ToolServer/ToolServerNode/core/tools/shell.py:27](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/tools/shell.py#L27)
- The node container mounts the shared ToolServer config directory read-write, so a hijacked session can persistently change extensions and container arguments for every later session. (ASI06, T1; C6) — [assets/config/manager.yml:19](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L19); [ToolServer/ToolServerNode/core/register/register.py:57-59](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/register/register.py#L57-L59)
- The model installs and runs arbitrary third-party packages as root without consent. (ASI04, T17; C7) — [ToolServer/ToolServerNode/core/tools/shell.py:28](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/tools/shell.py#L28)
- Long-lived service keys are readable from the agent's root shell via the mounted config volume. (ASI03, LLM02; C8) — [assets/config/node.yml:24](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/node.yml#L24); [assets/config/manager.yml:19](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L19)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

XAgent's tools run as root inside a privileged Docker container that sits on the same Docker network as the MySQL and MongoDB databases and the unauthenticated ToolServerManager that can start new containers. Nothing narrows what the agent's shell can reach or checks per-request authority. The web server's default credentials and access checks are not locked down. A hijacked agent effectively holds host root.

- **S L0:** Tool processes run as root in a privileged container with network reach to the databases. — [assets/config/manager.yml:16](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L16); [ToolServer/ToolServerNode/core/tools/shell.py:27](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/tools/shell.py#L27); [assets/config/manager.yml:15](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L15) (verified)
  - *To reach the next level:* No dedicated, role-scoped identity for tool execution; the agent's shell inherits container root plus database and manager access.
- **C L0:** No authorization layer exists between the model's tool choice and execution; execute_tool calls any registered tool with the model's arguments. — [ToolServer/ToolServerNode/main.py:260](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/main.py#L260); [XAgent/function_handler.py:214](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgent/function_handler.py#L214) (verified)
  - *To reach the next level:* No authorization check on any tool path.
- **D L0:** Default install runs nodes privileged; default web-server credentials are not locked down. — [assets/config/manager.yml:16](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L16) (verified)
  - *To reach the next level:* Least privilege requires manual hardening of manager.yml and compose.
- **B L0:** A privileged root container can take over the Docker host and has network reach to MySQL (all users' interactions and tokens). — [assets/config/manager.yml:16](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L16) (verified)
  - *To reach the next level:* Blast radius is host-equivalent and cross-tenant; nothing scopes it to one session.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no approval step before any tool runs. The default mode is fully automatic, and the optional 'manual' mode only lets the user edit the agent's previous thoughts or a subtask goal between steps; the next tool call is generated and executed without being shown for approval. The root shell, Python notebook, file writer and web fetch all execute directly.

- **S L0:** No human approval of tool calls exists; manual mode only exposes prior node data for editing. — searched `rg -n -i 'approv|confirm'` in `XAgent ToolServer/ToolServerNode/core ToolServer/ToolServerNode/main.py` → 0 hits (No approval or confirmation logic anywhere in the agent loop or tool server.); [XAgent/inner_loop_search_algorithms/ReACT.py:227](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgent/inner_loop_search_algorithms/ReACT.py#L227) (verified)
  - *To reach the next level:* No per-call approval showing the exact command before execution.
- **C L0:** The most powerful tool, a root bash shell, runs without any gate. — [ToolServer/ToolServerNode/core/tools/shell.py:27](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/tools/shell.py#L27); [XAgent/function_handler.py:214](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgent/function_handler.py#L214) (verified)
  - *To reach the next level:* Shell and code execution paths are not gated.
- **D L0:** Default mode is 'auto'; the interrupt path is opt-in. — [run.py:22](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/run.py#L22) (verified)
  - *To reach the next level:* Approval would need to be on by default.
- **B L0:** Root shell actions, file overwrites and outbound requests are irreversible and unbounded; no checkpoints or dry-runs. — [ToolServer/ToolServerNode/core/tools/shell.py:27](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/tools/shell.py#L27); [ToolServer/ToolServerNode/core/envs/web.py:104-106](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/envs/web.py#L104-L106) (verified)
  - *To reach the next level:* No checkpoints or rollback for workspace or external actions.
- **Cap:** C2-POWERBYPASS — The root shell tool (shell.py:27) executes via execute_tool (main.py:260) with no gate in the default auto mode (run.py:22).

### C3 Tool & action scoping — 0.15 (high)

The default tool set includes a raw root bash shell, a Python notebook and an arbitrary-URL web fetcher, all enabled by default. The file tools resolve paths and check containment, though the shell makes that moot anyway. The file containment check and the tool blacklist are not strict boundaries, and the web fetcher's request handling is not locked down.

- **S L1:** File tools use realpath-based containment (not a strict boundary); shell and notebook are raw passthrough; URLs are not restricted. — [ToolServer/ToolServerNode/core/tools/shell.py:56](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/tools/shell.py#L56) (verified)
  - *To reach the next level:* No allowlist validation: the shell takes an arbitrary string and browse_website fetches any URL including internal hosts.
- **C L1:** Only the filesystem env validates paths; shell, notebook and web do not. (verified)
  - *To reach the next level:* Most built-in tools lack validation.
- **D L0:** Shell, Python execution, file write and web fetch are all enabled by default; the tool blacklist is not a strict boundary. — [ToolServer/ToolServerNode/main.py:260](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/main.py#L260) (verified)
  - *To reach the next level:* No read-only default tool set; tools are not reliably disableable.
- **B L0:** A misused shell runs any command as root in a privileged container with network reach to internal services. — [ToolServer/ToolServerNode/core/tools/shell.py:27](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/tools/shell.py#L27); [assets/config/manager.yml:16](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L16) (verified)
  - *To reach the next level:* Tools are not scoped to the workspace or bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation — 0.25 (high)

All model-generated code and commands run inside a per-session ToolServerNode container rather than on the XAgent host, and there is no host fallback. But the container is started with privileged: true, runs as root, has Docker installed and started inside it, mounts the shared configuration directory read-write, and joins the network holding the databases and the container manager. A privileged container is not a meaningful boundary: an escape to the host is a documented, routine technique.

- **S L1:** Isolation is a privileged root container, weaker than a stock container with default capabilities. — [assets/config/manager.yml:16](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L16); [ToolServer/ToolServerNode/main.py:30](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/main.py#L30) (verified)
  - *To reach the next level:* Needs at least a non-privileged container; hardened profile (non-root, cap-drop, no-new-privileges, seccomp) for L3.
- **C L2:** Shell, notebook and code-interpreter tools all execute inside the node container via execute_tool; no exec path in the XAgent host process was found. — [ToolServer/ToolServerNode/main.py:260](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/main.py#L260); searched `rg -n 'subprocess|os.system|\bexec\(|\beval\(' --glob '*.py'` in `XAgent XAgentServer` → 0 hits (No model-driven exec on the XAgent host side.) (verified)
  - *To reach the next level:* Coverage is held at one level above the weak isolation primitive.
- **D L2:** The container path is the only execution path and the model cannot opt out of it, but its privileged policy comes from manager.yml, which the node itself can write. — [ToolServer/ToolServerManager/main.py:127-129](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerManager/main.py#L127-L129); [assets/config/manager.yml:19](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L19) (verified)
  - *To reach the next level:* Sandbox policy lives in a directory mounted read-write into the sandbox.
- **B L0:** Privileged root container with docker.io, a RW config mount and network access to databases and the manager. — [assets/config/manager.yml:16](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L16); [assets/config/manager.yml:19](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L19); [dockerfiles/ToolServerNode/Dockerfile:11](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/dockerfiles/ToolServerNode/Dockerfile#L11) (verified)
  - *To reach the next level:* Remove privileged, mount config read-only, isolate the node network, add resource limits.
- **Cap:** C4-HOSTROOT — The default node container is started with privileged: true (manager.yml:16) and runs commands as root (shell.py:27).

### C5 Untrusted input blast radius — 0.00 (high)

The agent browses arbitrary web pages and search results and feeds tool output back to the model as 'system' messages, giving them the same standing as its own instructions. Nothing detects, marks, or limits what a hijacked agent can do after reading that content. With a root shell, unrestricted egress and reach to the shared databases, an injected page can make the agent exfiltrate data and take irreversible actions with no human involved.

- **S L0:** No structural limit on a hijacked agent; no taint or provenance handling. — searched `rg -n -i 'untrusted|injection|sanitiz'` in `XAgent ToolServer/ToolServerNode` → 5 hits (All hits are the upload path-security helper's parameter names, not injection defenses.) (verified)
  - *To reach the next level:* No approval or capability removal after untrusted content is read.
- **C L0:** Tool results enter history with role 'system'. — [XAgent/function_handler.py:244](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgent/function_handler.py#L244) (verified)
  - *To reach the next level:* Untrusted tool and web results are not distinguished from instructions.
- **D L0:** No control exists to be on by default. — [XAgent/function_handler.py:244](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgent/function_handler.py#L244) (verified)
  - *To reach the next level:* No default control.
- **B L0:** Hijacked agent can exfiltrate via browse_website/shell and run destructive root commands unattended. — [ToolServer/ToolServerNode/core/envs/web.py:104-106](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/envs/web.py#L104-L106); [ToolServer/ToolServerNode/core/tools/shell.py:27](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/tools/shell.py#L27) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions both happen without a human.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.00 (high)

XAgent has no long-term memory store (the Pinecone vector DB is commented out). But the ToolServer configuration directory is bind-mounted read-write into every node container the agent controls as root. That directory holds node.yml, which is loaded by every future node and whose enabled_extensions list imports modules into the tool server, and manager.yml, which defines the docker run arguments for every new node. A single hijacked session can therefore persist changes that affect all later sessions and users, with no validation or review.

- **S L0:** The agent's root shell can rewrite node.yml/manager.yml, which are auto-loaded by later nodes and the manager and can add extensions or change container arguments. — [assets/config/manager.yml:19](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L19); [ToolServer/ToolServerNode/core/register/register.py:57-59](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/register/register.py#L57-L59); [ToolServer/ToolServerNode/config.py:20](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/config.py#L20) (verified)
  - *To reach the next level:* Config the sandbox can write must not be auto-loaded with security effect.
- **C L0:** No persistence path is controlled. — [assets/config/manager.yml:19](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L19) (verified)
  - *To reach the next level:* No integrity check on the shared config files.
- **D L0:** The config volume is shared by all nodes and users. — [docker-compose.yml:100-106](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/docker-compose.yml#L100-L106) (verified)
  - *To reach the next level:* No per-user or per-session isolation of the config the agent can write.
- **B L0:** Poisoned config persists across sessions and users and alters tool loading and container privileges. — [ToolServer/ToolServerNode/core/register/register.py:57-59](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/register/register.py#L57-L59); [assets/config/manager.yml:16](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L16) (verified)
  - *To reach the next level:* Changes are neither session-scoped nor reviewed.
- **Cap:** none

### C7 Third-party extensions — 0.00 (high)

The shell tool's own description tells the model to install packages, and it does so as root with no consent or pinning, so third-party code is fetched and run at the model's choice. Tool-server extensions are imported in-process from a config list that the agent's container can write. The optional XAgentGen local-model server loads models with trust_remote_code='auto', but it is not part of the default compose stack.

- **S L0:** Model-chosen package installs through the root shell; extensions imported by name with no verification. — [ToolServer/ToolServerNode/core/tools/shell.py:28](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/tools/shell.py#L28); [ToolServer/ToolServerNode/core/register/register.py:141](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/register/register.py#L141) (verified)
  - *To reach the next level:* No pinning or integrity checks on anything the agent installs or loads.
- **C L0:** No extension type is verified. — [ToolServer/ToolServerNode/core/register/register.py:141](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/register/register.py#L141) (verified)
  - *To reach the next level:* No verification for packages, extensions or models.
- **D L0:** Packages are installed whenever the model decides; extensions can be added via a config file the sandbox can write. — [assets/config/manager.yml:19](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L19); [ToolServer/ToolServerNode/core/register/register.py:57-59](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/register/register.py#L57-L59) (verified)
  - *To reach the next level:* Extensions should be explicit, operator-scoped installs.
- **B L0:** Extensions run in-process in the tool server; installed packages run as root in the privileged container. — [ToolServer/ToolServerNode/core/register/register.py:141-144](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/register/register.py#L141-L144); [assets/config/manager.yml:16](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L16) (verified)
  - *To reach the next level:* No per-extension confinement.
- **Cap:** C7-RCELOAD — By default the agent installs and runs model-chosen third-party packages as root (shell.py:28) without consent.

### C8 Secrets & sensitive-data protection — 0.12 (high)

Default deployment credentials are not locked down. Bing and RapidAPI keys live in node.yml, which is mounted into the container where the model's root shell can read it. The recorder masks api_key fields only in the console log; the full run config, including LLM API keys, is stored unmasked in the database, and the conversation-sharing path does not fully protect secrets either.

- **S L0:** Plaintext credentials and no secret handling beyond one log regex. (verified)
  - *To reach the next level:* Secrets need a secret store or at least type-level masking on main paths.
- **C L1:** Only the console log path masks api_key; DB records are unmasked. — [XAgent/recorder.py:90](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgent/recorder.py#L90); [XAgent/recorder.py:305](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgent/recorder.py#L305) (verified)
  - *To reach the next level:* Transcripts, DB records and subprocess-readable files are unprotected.
- **D L1:** No telemetry by default, but full records are logged by default and redaction covers only the api_key field in logs. — [XAgent/recorder.py:89-93](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgent/recorder.py#L89-L93) (verified)
  - *To reach the next level:* Verbose payload logging should be off or redacted by default.
- **B L0:** Long-lived service keys (Bing, RapidAPI) are reachable by the model's root shell. — [assets/config/node.yml:24](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/node.yml#L24); [assets/config/manager.yml:19](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L19) (verified)
  - *To reach the next level:* Keys reachable from the sandbox should be scoped and short-lived, or absent.
- **Cap:** none

### C9 Audit & traceability — 0.45 (high)

Every tool call that goes through the function handler is written as a structured record (tool name, input, output, status, the model's thoughts, timestamp) to MySQL, alongside LLM input/output pairs and plan changes. There is no human-versus-agent attribution or approval record because there are no approvals. The database is on the agent container's network, so the record is not protected from the agent.

- **S L2:** Structured per-tool-call records with arguments, outputs, status and create_time in MySQL. — [XAgent/recorder.py:188-212](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgent/recorder.py#L188-L212); [XAgent/recorder.py:104](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgent/recorder.py#L104) (verified)
  - *To reach the next level:* No actor attribution (approver, principal) or correlation across internal agents.
- **C L2:** All tool calls, LLM calls and plan refinements are recorded via handle_tool_call and the recorder. — [XAgent/function_handler.py:266-273](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgent/function_handler.py#L266-L273) (verified)
  - *To reach the next level:* No records of configuration changes made from inside the node or of credential use.
- **D L1:** On by default, but stored in a MySQL instance on the agent container's network. — [assets/config/manager.yml:15](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/manager.yml#L15) (verified)
  - *To reach the next level:* Record store must be unreachable and unwritable from the agent's tools.
- **B L2:** Records are committed per action and DB errors re-raise; the record is written after the tool has already run. — [XAgent/recorder.py:38-44](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgent/recorder.py#L38-L44); [XAgent/function_handler.py:266](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgent/function_handler.py#L266) (verified)
  - *To reach the next level:* Records are not written before high-risk actions, nor tamper-evident.
- **Cap:** none

### C10 Limits & kill switch — 0.45 (high)

Each subtask is capped at 15 tool steps, plan depth and width are bounded, and shell and notebook commands time out after 300 seconds. There is no token, cost or wall-clock cap for a run, and the client call that executes a tool has no timeout. Background shells started with run_async are not bounded and keep running until the container is stopped; idle nodes are stopped after 30 minutes. Stopping from the web UI sets a flag that the agent checks between steps, then exits.

- **S L2:** Iteration cap per subtask plus per-command timeouts; cooperative halt via a Redis flag. — [XAgent/inner_loop_search_algorithms/ReACT.py:220](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgent/inner_loop_search_algorithms/ReACT.py#L220); [ToolServer/ToolServerNode/core/tools/shell.py:67](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/tools/shell.py#L67); [XAgentServer/interaction.py:111-114](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgentServer/interaction.py#L111-L114) (verified)
  - *To reach the next level:* No token/cost or run wall-clock cap; no rate limits on side-effecting tools.
- **C L2:** Loop cap plus shell/notebook timeouts; background async shells escape them. — [ToolServer/ToolServerNode/core/tools/shell.py:93-108](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerNode/core/tools/shell.py#L93-L108); [XAgent/toolserver_interface.py:338](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/XAgent/toolserver_interface.py#L338) (verified)
  - *To reach the next level:* Background processes do not count against any budget.
- **D L2:** Sensible operator-configurable defaults in config.yml and node.yml. — [assets/config.yml:44](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config.yml#L44); [assets/config/node.yml:62](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/assets/config/node.yml#L62) (verified)
  - *To reach the next level:* node.yml limits sit in a directory the agent's container can write.
- **B L1:** No spend ceiling; background shells keep running after the loop until the node is stopped or idles out after 30 minutes. — [ToolServer/ToolServerManager/node_checker.py:52](https://github.com/openbmb/xagent/blob/0dea79924347fad46e114621d7334d1136b30691/ToolServer/ToolServerManager/node_checker.py#L52) (verified)
  - *To reach the next level:* Stopping should cancel in-flight calls and kill background processes; add a spend ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web search and browse_website (ToolServer/ToolServerNode/core/envs/web.py:103-106) · [B] sensitive data/systems: node.yml keys via RW config mount (assets/config/manager.yml:19) · [C] state change / egress: root shell (ToolServer/ToolServerNode/core/tools/shell.py:27) and arbitrary GET (web.py:104) · Same default session? Yes

## Highest-impact improvements
1. Remove privileged: true from the node creation_kwargs and run tools as a non-root user with dropped capabilities. — C4 S L1→L3, +0.150 before caps (Playbook 3, step 1)
2. Mount the ToolServer config into nodes read-only (or not at all) and keep secrets out of it. — C6 S L0→L3, +0.225 before caps (Playbook 2)
3. Put nodes on an isolated network with no route to MySQL, MongoDB, Redis or the manager. — C4 B L0→L2, +0.100 before caps (Playbook 3)
4. Add a per-call approval prompt showing the exact shell command, code or URL before execution, on by default. — C2 S L0→L3, +0.225 before caps (Playbook 5)
5. Generate all deployment secrets at install time. — C8 S L0→L1, +0.075 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- XAgentWeb (frontend) and XAgentGen (optional local-model server; trust_remote_code='auto' at XAgentGen/app.py:62) were not scored in depth.
- Container-escape impact of privileged mode is inferred from standard Docker behaviour, not tested.
- No text aimed at AI reviewers was found in the repository.
