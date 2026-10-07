# Defense-in-Depth Score: ChatDev 2.0 (DevAll)

**Repo:** https://github.com/openbmb/chatdev · **Commit:** `4fb2db0ea90375ce1059f44fe03ffbd191a7a169` · **Reviewed:** 2026-10-04
**What it is:** Zero-code multi-agent orchestration platform: YAML-defined agent workflows run by a FastAPI backend with a Vue web console.
**Category:** Agent Frameworks
**Scored configuration:** Backend and console started with `make dev` (server_main defaults) running the shipped yaml_instance workflows such as ChatDev_v1.yaml with default node settings.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions yes · sub agents yes · external communication no

## Score: 1.7 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L2 | L1 | L2 | L0 | 0.33 | — | **0.33** | High |
| C4 | Code-execution isolation | L2 | L3 | L0 | L0 | 0.38 | G1 | **0.38** (alt) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L0 | L0 | L1 | 0.12 | — | **0.12** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | C7-RCELOAD | **0.00** | High |
| C8 | Secrets & sensitive-data protection | L1 | L0 | L1 | L0 | 0.12 | — | **0.12** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L0 | 0.35 | — | **0.35** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


ChatDev 2.0 runs agent-written code, package installs and MCP servers directly on the host, with the provider API key in every subprocess and no approval step. The backend's default network exposure is not locked down. Only the file tools are meaningfully scoped. Run it only on an isolated machine or container, bound to localhost.

## Critical gaps
- Every code-execution path (execute_code, uv_run, python nodes, MCP stdio) runs as an unsandboxed host subprocess that inherits os.environ, including the provider API key loaded from .env. (ASI05, T11; C4) — [functions/function_calling/code_executor.py:38-47](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/code_executor.py#L38-L47); [functions/function_calling/uv_related.py:159-170](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/uv_related.py#L159-L170); [runtime/node/executor/python_executor.py:127-143](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/python_executor.py#L127-L143); [utils/env_loader.py:17-28](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/utils/env_loader.py#L17-L28)
- Content from web pages, files or MCP results can hijack an agent into exfiltrating the provider key and running irreversible host commands with no human in the loop. (ASI01, LLM01, T6; C5) — [functions/function_calling/web.py:129-168](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/web.py#L129-L168); [functions/function_calling/code_executor.py:38-47](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/code_executor.py#L38-L47); [utils/env_loader.py:17-28](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/utils/env_loader.py#L17-L28)
- Agents can install model-chosen Python packages (install hooks run on host), and MCP servers launch unpinned via uvx with the full environment. (ASI04, T17, LLM03; C7) — [functions/function_calling/uv_related.py:209-220](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/uv_related.py#L209-L220); [yaml_instance/blender_3d_builder_simple.yaml:220-224](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/yaml_instance/blender_3d_builder_simple.yaml#L220-L224)
- The long-lived provider API key from .env is placed in os.environ and inherited by every code-execution subprocess and MCP server, where model-written code can read it. (ASI03, LLM02; C8) — [utils/env_loader.py:17-28](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/utils/env_loader.py#L17-L28); [functions/function_calling/uv_related.py:159-170](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/uv_related.py#L159-L170); [runtime/node/agent/tool/tool_manager.py:508-518](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/agent/tool/tool_manager.py#L508-L518)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

ChatDev's backend acts with whatever authority the operating-system user who started it has, and the HTTP API's default exposure is not locked down. The provider API key from .env is loaded into the process environment and inherited by every code-running tool and local MCP server. There is no per-request or per-tool authorization layer, so a hijacked agent acts with the user's full account.

- **S L0:** Ambient OS-user authority plus the long-lived provider key in os.environ; no dedicated or scoped identity. — [utils/env_loader.py:17-28](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/utils/env_loader.py#L17-L28); [functions/function_calling/code_executor.py:38-47](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/code_executor.py#L38-L47) (verified)
  - *To reach the next level:* No scoped identity or deterministic authorization gate in front of tools.
- **C L0:** No authorization check exists in the tool executor, and HTTP API access control is not locked down; every subprocess receives the full environment. — [functions/function_calling/uv_related.py:159-170](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/uv_related.py#L159-L170); [runtime/node/executor/python_executor.py:127-143](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/python_executor.py#L127-L143) (verified)
  - *To reach the next level:* No authorization layer on any tool path.
- **D L0:** The default launch configuration does not lock down access, which exposes full-user authority. (verified)
  - *To reach the next level:* Default network exposure and access control should be hardened.
- **B L0:** A hijacked agent reaches arbitrary code execution as the user, the user's whole home directory, and the provider key. — [utils/function_manager.py:59-64](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/utils/function_manager.py#L59-L64) (verified)
  - *To reach the next level:* Authority is not narrowed to a project or tenant.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no approval step for any tool call. Agent nodes run every model-requested tool immediately, including code execution, package installation, recursive deletes and web fetches. The human node and the call_user tool let a workflow author or the model ask a person for input, but neither gates tool execution. Deletions in the workspace and actions taken by executed code are not reversible.

- **S L0:** No approval mechanism; tool calls go straight from the model response to execute_tool. — [runtime/node/executor/agent_executor.py:786-795](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/agent_executor.py#L786-L795); searched `rg -n -i approv -g '*.py'` in `runtime workflow server functions` → 0 hits (No approval concept anywhere in the tool path.) (verified)
  - *To reach the next level:* No per-call human approval showing the exact call.
- **C L0:** The most powerful tools (uv_run, execute_code, install_python_packages) are ungated like everything else. — [yaml_instance/ChatDev_v1.yaml:53-64](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/yaml_instance/ChatDev_v1.yaml#L53-L64); searched `rg -n -i approv -g '*.py'` in `runtime workflow server functions` → 0 hits (No approval concept anywhere in the tool path.) (verified)
  - *To reach the next level:* No gate on the exec tools or any other tool.
- **D L0:** No gate exists to be on by default. — searched `rg -n -i approv -g '*.py'` in `runtime workflow server functions` → 0 hits (No approval concept anywhere in the tool path.) (verified)
  - *To reach the next level:* Approval would need to be on by default.
- **B L0:** A wrong action can recursively delete workspace trees or run arbitrary host code with no checkpoint or undo. — [functions/function_calling/file.py:300-306](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/file.py#L300-L306); [functions/function_calling/code_executor.py:38-47](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/code_executor.py#L38-L47) (verified)
  - *To reach the next level:* No checkpoints or rollback for filesystem state.
- **Cap:** none

### C3 Tool & action scoping — 0.33 (high)

The built-in file tools are well scoped: every path is resolved and checked to stay inside the session's code workspace. That scoping is undone by the execution tools the flagship workflow also enables: execute_code and uv_run run arbitrary Python on the host with model-chosen arguments and environment variables, install_python_packages accepts model-chosen package specs including URLs, and read_webpage_content fetches any URL. Each agent node receives only the tools its YAML lists, but the shipped ChatDev workflow lists write, delete, and exec tools.

- **S L2:** File tools use resolved-path containment, but the exec tools are raw passthrough of code, args and env. — [functions/function_calling/file.py:62-67](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/file.py#L62-L67); [functions/function_calling/uv_related.py:159-170](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/uv_related.py#L159-L170); [functions/function_calling/web.py:129-168](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/web.py#L129-L168) (verified)
  - *To reach the next level:* Exec tools and the URL fetcher have no allowlist validation.
- **C L1:** Only the file tools validate; code, package, web and MCP tools do not. — [functions/function_calling/file.py:62-67](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/file.py#L62-L67); [functions/function_calling/code_executor.py:38-47](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/code_executor.py#L38-L47); [functions/function_calling/uv_related.py:209-220](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/uv_related.py#L209-L220) (verified)
  - *To reach the next level:* Most built-in tools, including exec and web tools, would need validation.
- **D L2:** Tools are selected per agent node in YAML, but the flagship ChatDev_v1 workflow grants uv_related:All plus delete/save tools. — [yaml_instance/ChatDev_v1.yaml:53-64](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/yaml_instance/ChatDev_v1.yaml#L53-L64); [runtime/node/executor/agent_executor.py:712-716](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/agent_executor.py#L712-L716) (verified)
  - *To reach the next level:* Shipped workflows grant write and exec by default.
- **B L0:** A misused uv_run or execute_code reaches the whole machine as the user. — [functions/function_calling/uv_related.py:159-170](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/uv_related.py#L159-L170); [functions/function_calling/code_executor.py:38-47](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/code_executor.py#L38-L47) (verified)
  - *To reach the next level:* Exec tools are not scoped to the workspace.
- **Cap:** none

### C4 Code-execution isolation — 0.38 (high)

All model-influenced code runs as a plain subprocess on the host: execute_code, uv_run, package installs, python nodes and local MCP servers. Each one inherits the server's full environment, including the provider API key, and has unrestricted network and filesystem access as the user. The optional Docker Compose setup runs the backend as a non-root user inside a container, but it bind-mounts the whole repo including .env, injects the same secrets, and leaves network open. It is also not the mode the README leads with.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Same-user host subprocess for every exec path. — [functions/function_calling/code_executor.py:38-47](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/code_executor.py#L38-L47); [runtime/node/executor/python_executor.py:127-143](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/python_executor.py#L127-L143); searched `rg -n -i 'sandbox|seccomp|nsjail|firejail|gvisor' -g '*.py'` in `.` → 0 hits (No isolation primitive in any Python source.) (verified)
    - *To reach the next level:* No OS-level separation in the default mode.
  - **C L0:** No path is isolated. — [functions/function_calling/uv_related.py:159-170](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/uv_related.py#L159-L170); [runtime/node/agent/tool/tool_manager.py:508-518](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/agent/tool/tool_manager.py#L508-L518) (verified)
    - *To reach the next level:* No execution path goes through a sandbox.
  - **D L0:** No isolation on by default. — searched `rg -n -i 'sandbox|seccomp|nsjail|firejail|gvisor' -g '*.py'` in `.` → 0 hits (No isolation primitive in any Python source.) (verified)
    - *To reach the next level:* Isolation would need to be on by default.
  - **B L0:** Host-equivalent: the user's files, full network, and API_KEY in every subprocess environment. — [utils/env_loader.py:17-28](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/utils/env_loader.py#L17-L28); [functions/function_calling/uv_related.py:159-170](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/uv_related.py#L159-L170) (verified)
    - *To reach the next level:* Credentials and host filesystem are reachable from executed code.
- **opt-in Docker Compose deployment** (alt; raw 0.38, cap G1 → 0.38) ← counted
  - **S L2:** Stock python:3.12-slim container running as a dedicated non-root appuser, no capability drops or seccomp profile. — [Dockerfile:56-57](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/Dockerfile#L56-L57) (verified)
    - *To reach the next level:* No dropped capabilities, no-new-privileges, or read-only rootfs.
  - **C L3:** The whole backend runs in the container, so every exec path, MCP stdio server and package install is inside it. — [Dockerfile:56-57](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/Dockerfile#L56-L57); [compose.yml:8-14](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/compose.yml#L8-L14) (verified)
    - *To reach the next level:* No fail-closed guarantee; nothing prevents running the backend on the host instead.
  - **D L0:** Opt-in; README presents it as an alternative to make dev. — [compose.yml:8-14](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/compose.yml#L8-L14) (verified)
    - *To reach the next level:* Not the default deployment.
  - **B L0:** Repo root (including .env) is bind-mounted read-write and env_file injects secrets; full network egress. — [compose.yml:8-14](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/compose.yml#L8-L14) (verified)
    - *To reach the next level:* Credentials and repo mount are reachable inside the container.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.00 (high)

Untrusted content enters agents through web search and page fetches, uploaded files, MCP tool results and descriptions, and other agents' messages, all with the same standing as the user's task. Nothing tracks where content came from or restricts what an agent may do after reading it. A prompt injection in a fetched page can make an agent run code that reads the API key and sends it out, delete workspace files, or install packages, with no human involved.

- **S L0:** No structural limit; tool results are appended to the conversation as ordinary messages. — [runtime/node/executor/agent_executor.py:786-795](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/agent_executor.py#L786-L795); [functions/function_calling/web.py:129-168](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/web.py#L129-L168) (verified)
  - *To reach the next level:* No approval or capability cut-off once untrusted content is read.
- **C L0:** Untrusted sources are not distinguished from the principal's instructions. — [functions/function_calling/web.py:129-168](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/web.py#L129-L168) (verified)
  - *To reach the next level:* Sources are not labelled or separated.
- **D L0:** No control to be on by default. — searched `rg -n -i approv -g '*.py'` in `runtime workflow server functions` → 0 hits (No approval concept anywhere in the tool path.) (verified)
  - *To reach the next level:* A control would need to be on by default.
- **B L0:** A hijacked agent can exfiltrate the API key via code exec or URL fetch and take irreversible actions unattended. — [functions/function_calling/code_executor.py:38-47](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/code_executor.py#L38-L47); [utils/env_loader.py:17-28](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/utils/env_loader.py#L17-L28); [functions/function_calling/file.py:300-306](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/file.py#L300-L306) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions need human approval.
- **Cap:** C5-WORSTCASE — B is L0: unattended secret exfiltration plus irreversible host actions in the default configuration.

### C6 Memory, context & configuration integrity — 0.12 (high)

Memory is opt-in per workflow, but when enabled every agent output is written to a JSON store automatically, with no validation, and re-injected later as a user-role message. Memory files are keyed only by the path the workflow sets, with no per-user namespace, and the server has no users. Separately, the exec tools can write anywhere the user can, including the functions/ directory, whose Python files are auto-loaded as tools. That lets poisoned content persist as code.

- **S L1:** Memory writes are logged via record_memory_operation but not validated; retrieved memory is inserted with USER role. — [runtime/node/executor/agent_executor.py:1253-1271](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/agent_executor.py#L1253-L1271); [runtime/node/executor/agent_executor.py:1228-1234](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/agent_executor.py#L1228-L1234); [runtime/node/executor/agent_executor.py:1275](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/agent_executor.py#L1275) (verified)
  - *To reach the next level:* Memory entries lack provenance and are not presented as data.
- **C L0:** No memory store or auto-loaded tool directory is controlled. — [workflow/graph.py:255-258](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/workflow/graph.py#L255-L258); [utils/function_manager.py:59-64](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/utils/function_manager.py#L59-L64) (verified)
  - *To reach the next level:* At least the main memory store would need gating.
- **D L0:** No per-user or per-session namespace; memory_path is a shared file. — [workflow/graph.py:255-258](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/workflow/graph.py#L255-L258) (verified)
  - *To reach the next level:* Per-user/session namespaces enforced in queries.
- **B L1:** Poisoned memory persists across runs of the workflow and can steer agents that hold exec tools. — [runtime/node/executor/agent_executor.py:1228-1234](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/agent_executor.py#L1228-L1234); [yaml_instance/ChatDev_v1.yaml:53-64](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/yaml_instance/ChatDev_v1.yaml#L53-L64) (verified)
  - *To reach the next level:* Persistence should only influence text or gated actions.
- **Cap:** none
- **Notes:** The memory subsystem is opt-in per workflow (the flagship ChatDev_v1 sets memories: []); the score rates it as shipped when enabled. G1 is not applied because the weak control (logging) is not itself opt-in, and the score is below the G1 ceiling regardless.

### C7 Third-party extensions — 0.00 (high)

Shipped workflows give agents install_python_packages, so the model picks which packages to install from PyPI or a URL, and their install hooks run on the host. Local MCP servers are launched with unpinned commands such as 'uvx blender-mcp' and inherit the full environment by default. Function tools are any Python file in functions/, executed in-process at load.

- **S L0:** Model-chosen package installs and unpinned uvx MCP launches. — [functions/function_calling/uv_related.py:209-220](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/uv_related.py#L209-L220); [yaml_instance/data_visualization_basic.yaml:104](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/yaml_instance/data_visualization_basic.yaml#L104); [yaml_instance/blender_3d_builder_simple.yaml:220-224](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/yaml_instance/blender_3d_builder_simple.yaml#L220-L224) (verified)
  - *To reach the next level:* Extensions would need at least user-chosen, pinned sources.
- **C L0:** No extension type is verified. — [utils/function_manager.py:59-64](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/utils/function_manager.py#L59-L64); [yaml_instance/blender_3d_builder_simple.yaml:220-224](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/yaml_instance/blender_3d_builder_simple.yaml#L220-L224) (verified)
  - *To reach the next level:* No verification on any extension type.
- **D L0:** Workflow YAML and tool files add extensions silently. (verified)
  - *To reach the next level:* Extensions should need explicit consent showing what will run.
- **B L0:** Function tools run in-process; MCP stdio servers inherit the full environment by default. — [utils/function_manager.py:59-64](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/utils/function_manager.py#L59-L64); [runtime/node/agent/tool/tool_manager.py:508-518](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/agent/tool/tool_manager.py#L508-L518); [entity/configs/node/tooling.py:431](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/entity/configs/node/tooling.py#L431) (verified)
  - *To reach the next level:* Extensions should run in a separate process with a scrubbed environment.
- **Cap:** C7-RCELOAD — By default agents can install model-chosen packages without consent.

### C8 Secrets & sensitive-data protection — 0.12 (high)

Provider keys come from a .env file loaded into os.environ. They do not appear in prompts. They are inherited by every subprocess the agents start, so model-written code can read them. No redaction exists anywhere: tool arguments and full results are logged to the session's execution_logs.json and streamed to the web console. No telemetry was found.

- **S L1:** Secrets come from env vars only; no masking or redaction exists. — [utils/env_loader.py:17-28](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/utils/env_loader.py#L17-L28); searched `rg -n -i 'redact|mask_secret|SecretStr' -g '*.py'` in `.` → 0 hits (No redaction helper anywhere.) (verified)
  - *To reach the next level:* No type-level masking or log filters.
- **C L0:** No path (logs, subprocess env, transcripts) is protected. — [utils/logger.py:248-262](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/utils/logger.py#L248-L262); [functions/function_calling/uv_related.py:159-170](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/uv_related.py#L159-L170) (verified)
  - *To reach the next level:* At least logs and transcripts would need redaction.
- **D L1:** No telemetry, but full tool payload logging is on by default and unredacted. — searched `rg -n -i 'sentry|posthog|telemetry' -g '*.py'` in `.` → 0 hits (No telemetry SDK.); [utils/logger.py:248-262](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/utils/logger.py#L248-L262) (verified)
  - *To reach the next level:* Payload logging should be minimised or redacted by default.
- **B L0:** The long-lived provider key is reachable by every subprocess and model-written code. — [utils/env_loader.py:17-28](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/utils/env_loader.py#L17-L28); [runtime/node/executor/python_executor.py:127-143](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/python_executor.py#L127-L143) (verified)
  - *To reach the next level:* Keys should not be present in exec subprocess environments.
- **Cap:** none

### C9 Audit & traceability — 0.35 (high)

Every tool call, built-in or MCP, is recorded before and after execution, with arguments, success flag, result and timing, tagged by node id. The record is held in memory and streamed to the web console. It is written to execution_logs.json in the session directory only after a workflow finishes successfully, so failed, cancelled or crashed runs lose it. The file sits where the agents' own exec tools can edit it, and nothing identifies who started a run.

- **S L2:** Structured BEFORE/AFTER records with arguments, results and timing for every tool call. — [runtime/node/executor/agent_executor.py:778-785](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/agent_executor.py#L778-L785); [utils/logger.py:248-262](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/utils/logger.py#L248-L262) (verified)
  - *To reach the next level:* No actor attribution (requesting principal) or correlation across subgraphs.
- **C L2:** All function, MCP and skill tool calls pass through record_tool_call; there are no approvals to record. — [runtime/node/executor/agent_executor.py:778-785](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/agent_executor.py#L778-L785); [runtime/node/executor/agent_executor.py:1275](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/agent_executor.py#L1275) (verified)
  - *To reach the next level:* Approvals, denials and config changes are not recorded.
- **D L1:** On by default, stored in the session directory writable by the agents' host-level exec tools. — [workflow/runtime/result_archiver.py:31-32](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/workflow/runtime/result_archiver.py#L31-L32) (verified)
  - *To reach the next level:* Record should be outside anything the agent's process can alter.
- **B L0:** Logs are persisted only at successful completion; failed or crashed runs lose the record. — [workflow/graph.py:336-340](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/workflow/graph.py#L336-L340); [workflow/runtime/result_archiver.py:31-32](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/workflow/runtime/result_archiver.py#L31-L32) (verified)
  - *To reach the next level:* Records should be flushed per action.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

Each agent node stops after 50 tool-call rounds, graph cycles stop after 100 iterations by default, and python nodes time out after 60 seconds. There is no token or cost ceiling and no wall-clock limit for a run. The model sets the timeout itself for execute_code and uv_run. Cancelling from the console sets a flag that is checked between steps and does not interrupt running subprocesses.

- **S L2:** Iteration caps plus per-execution subprocess timeouts, enforced in code. — [runtime/node/executor/agent_executor.py:1061-1069](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/agent_executor.py#L1061-L1069); [entity/configs/node/python_runner.py:27](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/entity/configs/node/python_runner.py#L27); searched `rg -n -i 'cost_limit|max_cost|token_budget|budget' -g '*.py'` in `runtime workflow server` → 0 hits (No token or cost ceiling.) (verified)
  - *To reach the next level:* No token/cost cap or run wall-clock limit.
- **C L2:** Top-level tool loop and cycles are capped and subprocesses time out; subgraphs and parallel nodes get fresh per-node budgets. — [runtime/node/executor/agent_executor.py:1061-1069](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/agent_executor.py#L1061-L1069); [workflow/cycle_manager.py:23](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/workflow/cycle_manager.py#L23) (verified)
  - *To reach the next level:* Sub-agents and parallel nodes don't share one budget.
- **D L1:** Defaults exist, but the model chooses the timeout argument for execute_code and uv_run. — [functions/function_calling/code_executor.py:1](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/code_executor.py#L1); [functions/function_calling/uv_related.py:277-283](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/functions/function_calling/uv_related.py#L277-L283) (verified)
  - *To reach the next level:* The model should not be able to raise its own limits.
- **B L1:** Ceilings are large (50 rounds x nodes x 100 cycles, unlimited spend) and cancel is cooperative, leaving in-flight subprocesses running. — [runtime/node/executor/base.py:143-146](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/runtime/node/executor/base.py#L143-L146); [workflow/cycle_manager.py:23](https://github.com/openbmb/chatdev/blob/4fb2db0ea90375ce1059f44fe03ffbd191a7a169/workflow/cycle_manager.py#L23) (verified)
  - *To reach the next level:* Tight cost ceilings and cancellation of pending calls.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web fetch/search and MCP results enter context (functions/function_calling/web.py:129) · [B] sensitive data/systems: provider API_KEY in os.environ inherited by subprocesses (utils/env_loader.py:28) · [C] state change / egress: unsandboxed code exec and package installs (functions/function_calling/uv_related.py:160) · Same default session? Yes

## Highest-impact improvements
1. Harden the HTTP API's default network exposure and access control. — C1 D L0→L2, +0.100 before caps (Playbook 4)
2. Pass a scrubbed environment (no API keys) to execute_code, uv_run, python nodes and MCP stdio servers; default inherit_env to false. — C8 B L0→L2, +0.100 before caps (Playbook 4)
3. Run code execution tools in a hardened per-session container with no secrets and restricted egress, failing closed when unavailable. — C4 S L0→L3, +0.225 before caps (Playbook 3 step 1)
4. Add a per-call human approval gate for exec, install, delete and egress tools that shows the exact arguments. — C2 S L0→L3, +0.225 before caps (Playbook 5)
5. Flush execution logs per action to a location outside the session workspace, including failed and cancelled runs. — C9 B L0→L2, +0.100 before caps (Playbook 1 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The frontend (Vue console), the Python SDK (runtime/sdk.py) and the legacy ChatDev 1.0 branch were not examined in depth.
- The Docker Compose deployment was scored only as an opt-in alternative for C4.
- No text aimed at AI reviewers was found in the repository.
