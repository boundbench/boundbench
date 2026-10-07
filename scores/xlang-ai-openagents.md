# Defense-in-Depth Score: OpenAgents (XLang)

**Repo:** https://github.com/xlang-ai/openagents · **Commit:** `ff2e46440699af1324eb25655b622c4a131265bb` · **Reviewed:** 2026-10-04
**What it is:** Self-hosted web platform with a data agent (Python/SQL code interpreter), a plugins agent (200+ third-party API plugins), and a web-browsing agent driven through a Chrome extension.
**Category:** AI Assistants
**Scored configuration:** Shipped docker-compose.yml: Flask backend on 0.0.0.0:8000 with CODE_EXECUTION_MODE=local, the code-interpreter sandbox commented out, and the shipped Next.js frontend defaults (Python tool selected).
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents no · external communication yes

## Score: 1.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | C2-POWERBYPASS | **0.00** | High |
| C3 | Tool & action scoping | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C4 | Code-execution isolation | L2 | L2 | L0 | L2 | 0.40 | G1 | **0.40** (alt) | Low |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L0 | L0 | L0 | 0.07 | — | **0.07** | High |
| C7 | Third-party extensions | L1 | L0 | L0 | L0 | 0.07 | C7-RCELOAD | **0.07** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C9 | Audit & traceability | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |


As shipped, OpenAgents runs model-written Python directly inside its backend process, with the operator's OpenAI key in the environment and no approval step. Any content that hijacks the model can execute code as root in the container and read every user's data; access control on the backend API is not locked down. The optional Docker code interpreter helps, but it is off by default and not hardened in this repo.

## Critical gaps
- Model-generated Python runs in-process in the backend by default. (ASI05; C4) — [docker-compose.yml:54](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/docker-compose.yml#L54); [real_agents/data_agent/evaluation/python_evaluator.py:96-101](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/data_agent/evaluation/python_evaluator.py#L96-L101)
- Code execution (the most powerful path) has no approval gate. (ASI02, ASI09; C2) — [real_agents/data_agent/evaluation/python_evaluator.py:96-101](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/data_agent/evaluation/python_evaluator.py#L96-L101)
- Hijacked sessions can exfiltrate secrets and take irreversible actions unattended. (ASI01; C5) — [real_agents/data_agent/evaluation/python_evaluator.py:96-101](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/data_agent/evaluation/python_evaluator.py#L96-L101)
- Unpinned Hugging Face model is downloaded and deserialized in-process at startup; plugins run in-process. (ASI04; C7) — [real_agents/plugins_agent/plugins/tool_selector.py:69-77](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/plugins_agent/plugins/tool_selector.py#L69-L77); [backend/api/chat_plugin.py:36](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/api/chat_plugin.py#L36); [real_agents/plugins_agent/plugins/utils.py:21-26](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/plugins_agent/plugins/utils.py#L21-L26)

## Criterion details

### C1 Identity & least privilege — 0.05 (high)

Access control on the backend API is not locked down. Every tool runs with the operator's own OpenAI key, Kaggle key, unauthenticated MongoDB/Redis, and any third-party plugin API keys users saved, all as root in the backend container. There is no per-request authorization of tool use.

- **S L0:** All tools act with the operator's ambient credentials (OPENAI_API_KEY in the backend env, Kaggle key baked into the image, unauthenticated MongoDB); no scoped identity exists. — [docker-compose.yml:49](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/docker-compose.yml#L49); [Dockerfile:188-192](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/Dockerfile#L188-L192); [backend/utils/user_conversation_storage.py:8](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/utils/user_conversation_storage.py#L8) (verified)
  - *To reach the next level:* No dedicated or role-scoped identity for the agent or its tools.
- **C L0:** No authorization layer on any tool or API path; API access control is not locked down. (verified)
  - *To reach the next level:* No authorization check on any tool or API path.
- **D L0:** Default deployment exposes the backend on all interfaces, and its access control is not locked down. — [Dockerfile:194](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/Dockerfile#L194); [docker-compose.yml:41-42](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/docker-compose.yml#L41-L42) (verified)
  - *To reach the next level:* Least privilege and authentication require manual hardening outside the project.
- **B L1:** A hijacked agent gets the OpenAI key (spend), the Kaggle account, all stored conversations and files, and saved plugin keys (e.g. Zapier, which can act across many connected apps). — [real_agents/plugins_agent/plugins/zapier/paths/__init__.py:1-6](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/plugins_agent/plugins/zapier/paths/__init__.py#L1-L6); [docker-compose.yml:49](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/docker-compose.yml#L49) (verified)
  - *To reach the next level:* Credentials span several systems with write capability; nothing is scoped to one project or read-only.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no human approval step anywhere in the backend. Model-generated Python and SQL run as soon as the model emits them, plugin API calls (including Zapier actions and a Netlify deploy endpoint) fire directly, and the web agent returns click and type actions for the user's live browser without a confirmation step in the server. The most powerful action, arbitrary code execution, is ungated by default.

- **S L0:** No approval mechanism exists; tool calls execute directly from the agent loop. — searched `rg -n -i 'approv|confirm' --type py` in `backend real_agents/adapters real_agents/data_agent real_agents/web_agent` → 1 hits (the single hit is an OpenAI model kwargs warning string (adapters/models/openai.py:155), not a gate); [real_agents/data_agent/evaluation/python_evaluator.py:96-101](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/data_agent/evaluation/python_evaluator.py#L96-L101) (verified)
  - *To reach the next level:* No per-call human approval of any kind.
- **C L0:** The most powerful tool (PythonCodeBuilder, in-process exec) is exempt along with SQL, plugins and webot actions. — [backend/api/chat_copilot.py:261-265](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/api/chat_copilot.py#L261-L265); [real_agents/plugins_agent/api_calling/base.py:101-115](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/plugins_agent/api_calling/base.py#L101-L115); [backend/api/webot_actions.py:9-20](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/api/webot_actions.py#L9-L20) (verified)
  - *To reach the next level:* Code execution and every other consequential tool bypass any gate (none exists).
- **D L0:** Nothing to enable; there is no approval setting. — searched `rg -n -i 'approv|confirm' --type py` in `backend real_agents/adapters real_agents/data_agent real_agents/web_agent` → 1 hits (the single hit is an OpenAI model kwargs warning string (adapters/models/openai.py:155), not a gate) (verified)
  - *To reach the next level:* Approval is not available even as an opt-in.
- **B L0:** Wrong actions are irreversible: arbitrary file deletion in the backend container, external API writes (Zapier, Netlify deploy), and clicks/form fills in the user's real browser sessions. — [real_agents/data_agent/evaluation/python_evaluator.py:96-101](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/data_agent/evaluation/python_evaluator.py#L96-L101); [real_agents/plugins_agent/plugins/Netlify/paths/deploy.py:6](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/plugins_agent/plugins/Netlify/paths/deploy.py#L6); [real_agents/web_agent/web_browsing/end2end/base.py:93-97](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/web_agent/web_browsing/end2end/base.py#L93-L97) (verified)
  - *To reach the next level:* No checkpoints, undo, or dry-run for any action.
- **Cap:** C2-POWERBYPASS — In-process Python execution (the most powerful path) runs with no gate in the default config (python_evaluator.py:96-101).

### C3 Tool & action scoping — 0.25 (high)

The data agent's main tools accept arbitrary model-written Python and SQL with no argument validation. Plugins are narrower: each one only calls fixed endpoints on a fixed host, and the model's chosen endpoint must exist in the plugin's table; web-agent actions are type-checked. Python is selected by default in the shipped frontend, and the tool set is chosen per request by the client rather than restricted to read-only.

- **S L1:** Python and SQL are raw passthrough; plugins restrict calls to a fixed endpoint table on hard-coded hosts, and webot args are type-checked. — [real_agents/data_agent/evaluation/python_evaluator.py:96-101](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/data_agent/evaluation/python_evaluator.py#L96-L101); [real_agents/adapters/schema.py:39](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/adapters/schema.py#L39); [real_agents/plugins_agent/api_calling/base.py:101-115](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/plugins_agent/api_calling/base.py#L101-L115) (verified)
  - *To reach the next level:* No allowlist validation of paths, URLs or queries on the exec/SQL tools.
- **C L1:** Only plugin endpoint lookup and webot argument typing validate input; the code and SQL tools do not. — [real_agents/plugins_agent/api_calling/base.py:101-115](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/plugins_agent/api_calling/base.py#L101-L115); [real_agents/web_agent/web_browsing/end2end/base.py:93-97](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/web_agent/web_browsing/end2end/base.py#L93-L97) (verified)
  - *To reach the next level:* Most built-in tools (Python, Echarts, SQL, Kaggle) have no input validation.
- **D L2:** Tools are selectable per request, but the default selection includes Python execution. — [backend/api/chat_copilot.py:301-304](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/api/chat_copilot.py#L301-L304); [frontend/pages/api/home/home.state.tsx:108](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/frontend/pages/api/home/home.state.tsx#L108) (verified)
  - *To reach the next level:* Default tool set is not read-only; exec is on by default.
- **B L0:** A misused Python tool reaches the whole backend container: filesystem, network, env credentials, all users' data. — [real_agents/data_agent/evaluation/python_evaluator.py:96-101](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/data_agent/evaluation/python_evaluator.py#L96-L101) (verified)
  - *To reach the next level:* Tools are general-purpose with no workspace or quantity bounds.
- **Cap:** none

### C4 Code-execution isolation — 0.40 (low)

In the default configuration (CODE_EXECUTION_MODE=local, set in both app.py and docker-compose.yml) model-generated Python runs through IPython's run_cell in the backend's own worker process. A code-safety check exists but is not a strict boundary. The code therefore runs as root in the backend container with the OpenAI key in its environment, full network access, unauthenticated MongoDB/Redis, and write access to the application source and every user's files. An optional Docker code-interpreter mode sends code to a separate kernel container, but it is commented out in the compose file, uses an image not in this repo, adds NET_ADMIN/SYS_PTRACE capabilities, and shares the backend data volume.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Code runs in-process via IPython InteractiveShell.run_cell; the code-safety check is not a strict boundary. — [real_agents/data_agent/evaluation/python_evaluator.py:96-101](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/data_agent/evaluation/python_evaluator.py#L96-L101) (verified)
    - *To reach the next level:* No isolation primitive at all on the default path.
  - **C L0:** The main exec path is unsandboxed; SQL and runtime pip installs also run on the backend. — [real_agents/data_agent/evaluation/python_evaluator.py:96-101](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/data_agent/evaluation/python_evaluator.py#L96-L101); [real_agents/adapters/schema.py:39](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/adapters/schema.py#L39); [real_agents/adapters/data_model/templates/skg_templates/table_templates.py:26-30](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/adapters/data_model/templates/skg_templates/table_templates.py#L26-L30) (verified)
    - *To reach the next level:* The main exec tool is not sandboxed.
  - **D L0:** Local, unsandboxed execution is the default in both the code and the shipped compose file. — [docker-compose.yml:54](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/docker-compose.yml#L54); [backend/app.py:11](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/app.py#L11) (verified)
    - *To reach the next level:* Sandboxing is off by default.
  - **B L0:** Executed code shares the backend process environment (OPENAI_API_KEY), runs as root with full egress, and can read/modify Mongo, Redis, app source and all users' uploads. — [docker-compose.yml:49](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/docker-compose.yml#L49); [backend/utils/user_conversation_storage.py:8](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/utils/user_conversation_storage.py#L8); [Dockerfile:188-192](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/Dockerfile#L188-L192) (verified)
    - *To reach the next level:* Credentials and host-equivalent access are reachable from executed code.
- **opt-in Docker code interpreter (CODE_EXECUTION_MODE=docker)** (alt; raw 0.40, cap G1 → 0.40) ← counted
  - **S L2:** Code is posted to a remote Jupyter kernel in a separate container; the image (xlanglab/xlang-code-interpreter-python) is not in this repo, so its hardening cannot be verified, and the compose stanza adds capabilities. — [real_agents/data_agent/evaluation/python_evaluator.py:222-224](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/data_agent/evaluation/python_evaluator.py#L222-L224); [docker-compose.yml:27-36](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/docker-compose.yml#L27-L36) (inferred)
    - *To reach the next level:* No verified non-root, seccomp or dropped-capability hardening.
  - **C L2:** Python and Echarts go to the kernel; SQL execution and pip installs still run in the backend. — [real_agents/data_agent/evaluation/python_evaluator.py:222-224](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/data_agent/evaluation/python_evaluator.py#L222-L224); [real_agents/adapters/schema.py:39](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/adapters/schema.py#L39) (inferred)
    - *To reach the next level:* SQL and runtime installs remain on the backend host process.
  - **D L0:** The interpreter service is commented out and the mode is opt-in. — [docker-compose.yml:54](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/docker-compose.yml#L54) (verified)
    - *To reach the next level:* Not enabled by default.
  - **B L2:** Sandbox mounts the shared backend data volume (all users' files) read-write, has unrestricted network plus NET_ADMIN, but per-kernel CPU/RAM/time limits are configured via env and no OpenAI key is passed. — [docker-compose.yml:32](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/docker-compose.yml#L32); [real_agents/data_agent/.code_interpreter_docker_env:5](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/data_agent/.code_interpreter_docker_env#L5); [docker-compose.yml:27-36](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/docker-compose.yml#L27-L36) (inferred)
    - *To reach the next level:* Workspace-wide shared mount and open egress remain.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.00 (high)

The agents read plenty of content their user did not write: uploaded files and databases, Kaggle datasets, third-party plugin API responses, and arbitrary web pages in the web agent. None of it is marked or treated differently; tool results go straight back into the model context. A hijacked session can run arbitrary Python with network access and the OpenAI key, call write-capable plugins, and drive the user's browser, with no human in the loop; frontend output rendering is not hardened either.

- **S L0:** Nothing limits a hijacked agent; no provenance, taint, or approval tied to untrusted content. — searched `rg -n -i 'untrusted|inject|sanitiz|provenance' --type py` in `backend real_agents/adapters real_agents/data_agent real_agents/web_agent` → 0 hits; [real_agents/data_agent/evaluation/python_evaluator.py:96-101](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/data_agent/evaluation/python_evaluator.py#L96-L101) (verified)
  - *To reach the next level:* No capability is disabled or gated after untrusted content is read.
- **C L0:** Untrusted sources (files, plugin outputs, web HTML) enter context exactly like user input. — [backend/api/webot_actions.py:9-20](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/api/webot_actions.py#L9-L20); [real_agents/plugins_agent/api_calling/base.py:101-115](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/plugins_agent/api_calling/base.py#L101-L115) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished at all.
- **D L0:** No control exists to be on by default. — searched `rg -n -i 'untrusted|inject|sanitiz|provenance' --type py` in `backend real_agents/adapters real_agents/data_agent real_agents/web_agent` → 0 hits (verified)
  - *To reach the next level:* No untrusted-input control exists.
- **B L0:** Unattended exfiltration (Python network egress) plus irreversible actions (code exec, Zapier/Netlify writes, browser clicks). — [real_agents/data_agent/evaluation/python_evaluator.py:96-101](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/data_agent/evaluation/python_evaluator.py#L96-L101) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions both happen with no human approval.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.07 (high)

Conversation history, including tool outputs that may carry injected text, is stored in MongoDB/Redis and reloaded as chat memory each time the conversation continues, with no validation or provenance. Uploaded and generated files persist in the backend data folder and can be reused as grounding data later. Per-user isolation of stored history and files is not a complete boundary. There are no auto-loaded instruction or config files from a workspace.

- **S L1:** History and files persist and are reloaded verbatim as conversation memory; writes are recorded but never validated. — [backend/api/chat_copilot.py:401-402](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/api/chat_copilot.py#L401-L402); [backend/utils/streaming.py:414-430](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/utils/streaming.py#L414-L430) (verified)
  - *To reach the next level:* Persisted entries carry no provenance and are not presented as data.
- **C L0:** No persistence path (history, grounding files, kernel state) is controlled. — [backend/api/chat_copilot.py:401-402](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/api/chat_copilot.py#L401-L402) (verified)
  - *To reach the next level:* No memory store has any write or load control.
- **D L0:** Per-user isolation of stored history is not a complete boundary. (verified)
  - *To reach the next level:* No enforced per-user isolation by default.
- **B L0:** Poisoned history or files persist across sessions (per-user isolation is not a complete boundary) and can steer code-execution tool calls. — [backend/api/chat_copilot.py:401-402](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/api/chat_copilot.py#L401-L402) (verified)
  - *To reach the next level:* Poisoned context persists and can trigger tool use.
- **Cap:** none

### C7 Third-party extensions — 0.07 (high)

The 232 plugins are vendored Python modules shipped in the repository and loaded in-process with importlib; they are part of the pinned code rather than downloaded at runtime, though each one sends data to a third-party API. At startup the plugin selector downloads the hkunlp/instructor-large embedding model from Hugging Face with no pinned revision or hash, and (through the sentence-transformers/InstructorEmbedding loader) deserializes its weights in the backend process. Runtime pip installs of helper packages are also unpinned. Nothing runs third-party code in a separate or confined process.

- **S L1:** The embedding model source is fixed by the project but unpinned (no revision or digest); pip installs at runtime are unpinned. — [real_agents/plugins_agent/plugins/tool_selector.py:69-77](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/plugins_agent/plugins/tool_selector.py#L69-L77); [real_agents/adapters/data_model/templates/skg_templates/table_templates.py:26-30](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/adapters/data_model/templates/skg_templates/table_templates.py#L26-L30) (verified)
  - *To reach the next level:* No version pin or integrity check on the downloaded model or packages.
- **C L0:** No extension type (model, packages, plugins) is verified. — [real_agents/plugins_agent/plugins/tool_selector.py:69-77](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/plugins_agent/plugins/tool_selector.py#L69-L77); [real_agents/plugins_agent/plugins/utils.py:21-26](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/plugins_agent/plugins/utils.py#L21-L26) (verified)
  - *To reach the next level:* No verification on any extension type.
- **D L0:** The model download happens automatically at import with no consent step. — [backend/api/chat_plugin.py:36](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/api/chat_plugin.py#L36) (verified)
  - *To reach the next level:* Third-party components are enabled automatically.
- **B L0:** Plugins, model weights and installed packages all run in-process with the backend's credentials and full environment. — [real_agents/plugins_agent/plugins/utils.py:21-26](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/plugins_agent/plugins/utils.py#L21-L26); [backend/api/chat_plugin.py:36](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/api/chat_plugin.py#L36) (verified)
  - *To reach the next level:* No separate process or scrubbed environment for third-party code.
- **Cap:** C7-RCELOAD — At import the backend auto-downloads an unpinned Hugging Face model (tool_selector.py:69-77, chat_plugin.py:36) whose loader deserializes pickle-format weights without consent (torch.load behaviour of the sentence-transformers/InstructorEmbedding stack, inferred).

### C8 Secrets & sensitive-data protection — 0.15 (high)

Secrets come from environment variables and files: the OpenAI key is set in the compose file, the Kaggle key is written into the image at build time, and third-party plugin API keys are stored in plaintext in Redis/MongoDB. The backend API's handling of stored plugin keys is not locked down. Full request bodies and generated code are logged at DEBUG/TRACE by default, and the only protection is loguru's diagnose=False on file sinks. Model-run code can read every secret from the process environment.

- **S L1:** Secrets are read from env vars and plaintext stores; the only masking is diagnose=False on log sinks, which suppresses variable values in tracebacks. — [backend/utils/utils.py:297](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/utils/utils.py#L297); [backend/api/tool.py:66-67](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/api/tool.py#L66-L67); searched `rg -n -i 'redact|mask|SecretStr' --type py` in `backend real_agents/adapters real_agents/data_agent` → 0 hits (verified)
  - *To reach the next level:* No type-level masking or log filters; plugin keys stored in plaintext.
- **C L1:** Only the error-traceback path is partially protected; request logs and the exec environment are not. — [backend/utils/utils.py:297](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/utils/utils.py#L297) (verified)
  - *To reach the next level:* Logs and subprocess/exec environments are unprotected.
- **D L0:** Verbose payload logging is on by default: TRACE to stdout and DEBUG request JSON to runtime.log. — [backend/utils/utils.py:292-305](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/utils/utils.py#L292-L305); [backend/api/chat_copilot.py:341-342](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/api/chat_copilot.py#L341-L342); [real_agents/data_agent/python/base.py:124](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/data_agent/python/base.py#L124) (verified)
  - *To reach the next level:* Payload logging is on by default.
- **B L0:** Long-lived OpenAI, Kaggle and plugin keys are reachable by model-written code in-process. — [docker-compose.yml:49](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/docker-compose.yml#L49); [real_agents/data_agent/evaluation/python_evaluator.py:96-101](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/data_agent/evaluation/python_evaluator.py#L96-L101) (verified)
  - *To reach the next level:* High-privilege keys are reachable by the model.
- **Cap:** none

### C9 Audit & traceability — 0.33 (high)

Each finished chat turn is written to MongoDB with the intermediate steps (tool used, generated code, results) and the final answer, and loguru writes runtime logs. This gives a usable transcript, but it is written only when the stream completes, carries no real actor identity, and lives in an unauthenticated database and log directory that the model's own code can modify.

- **S L1:** Per-turn records in Mongo include intermediate steps with generated code and results, but carry no timestamps or per-call status fields. — [backend/utils/streaming.py:414-430](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/utils/streaming.py#L414-L430) (verified)
  - *To reach the next level:* No structured per-tool-call record with timestamps and result status.
- **C L2:** Copilot, plugin and webot turns are all recorded through the streaming helpers. — [backend/utils/streaming.py:414-430](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/utils/streaming.py#L414-L430); [backend/utils/streaming.py:586](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/utils/streaming.py#L586) (verified)
  - *To reach the next level:* No record of approvals/denials (none exist) or configuration changes.
- **D L1:** On by default, but stored in an unauthenticated MongoDB and a .logging directory reachable by in-process model code. — [backend/utils/user_conversation_storage.py:8](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/utils/user_conversation_storage.py#L8); [backend/main.py:17-22](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/main.py#L17-L22) (verified)
  - *To reach the next level:* Records are writable by the agent's own code execution.
- **B L1:** Records are written after the stream ends; a crash, timeout or kill can lose the turn. — [backend/utils/streaming.py:414-430](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/utils/streaming.py#L414-L430); [backend/utils/streaming.py:277-280](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/utils/streaming.py#L277-L280) (verified)
  - *To reach the next level:* Records are not flushed per action.
- **Cap:** none

### C10 Limits & kill switch — 0.50 (high)

The agent loop stops after 5 iterations by default and runs in a child process that is terminated after a period without output (90 s for the data agent) or when the user hits stop. There is no wall-clock limit on the loop itself, no token or cost budget, no rate limiting, and the web agent's step loop is driven by the browser extension with no server-side cap. Termination kills the worker process but not anything it spawned.

- **S L2:** Iteration cap of 5 plus an idle timeout enforced by terminating the worker process. — [real_agents/adapters/agent_helpers/agent.py:428-429](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/adapters/agent_helpers/agent.py#L428-L429); [backend/schemas.py:3](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/schemas.py#L3); [backend/utils/streaming.py:277-280](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/utils/streaming.py#L277-L280) (verified)
  - *To reach the next level:* No token/cost cap or rate limit on side-effecting tools.
- **C L2:** Limits cover the agent loop and, in local mode, in-flight code execution inside the worker process; the webot loop has no step cap. — [backend/utils/streaming.py:230](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/utils/streaming.py#L230); [backend/api/webot_actions.py:9-20](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/api/webot_actions.py#L9-L20) (verified)
  - *To reach the next level:* Webot steps and processes spawned by executed code are not covered.
- **D L2:** Sensible defaults hard-coded; max_execution_time is None. — [real_agents/adapters/agent_helpers/agent.py:428-429](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/real_agents/adapters/agent_helpers/agent.py#L428-L429) (verified)
  - *To reach the next level:* Model-written code runs in the same process as the loop and could alter its limits.
- **B L2:** Stop terminates the worker process; children spawned by executed code and external plugin actions already sent are not undone. — [backend/utils/threading.py:35-44](https://github.com/xlang-ai/openagents/blob/ff2e46440699af1324eb25655b622c4a131265bb/backend/utils/threading.py#L35-L44) (verified)
  - *To reach the next level:* No spend ceiling and orphaned subprocesses can survive a stop.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: uploaded files, Kaggle data, plugin API responses, web pages (webot_actions.py:18-20, api_calling/base.py:101-115) · [B] sensitive data/systems: OPENAI_API_KEY in backend env (docker-compose.yml:49), stored plugin keys, all users' conversations · [C] state change / egress: in-process Python exec with network (python_evaluator.py:96-101), Zapier/Netlify plugins, browser actions · Same default session? Yes

## Highest-impact improvements
1. Ship the Docker code interpreter enabled by default (CODE_EXECUTION_MODE=docker) and fail closed if it is unavailable. — C4 D L0→L2, +0.100 before caps (Playbook 3 step 1)
2. Harden backend API access control. — C1 C L0→L2, +0.150 before caps (Playbook 4)
3. Require per-call user approval showing the exact generated code before PythonCodeBuilder/SQL execution. — C2 S L0→L3, +0.225 before caps (Playbook 5)
4. Harden handling of stored plugin API keys and stop logging full request JSON at DEBUG. — C8 D L0→L2, +0.100 before caps
5. Pin the Hugging Face model revision and load safetensors only. — C7 S L1→L2, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of commit ff2e464 only; nothing was executed, installed, or probed.
- The prebuilt Chrome extension (frontend/webot_extension.zip) and the external xlanglab/xlang-code-interpreter-python image were not examined; any confirmation step inside the extension is not credited.
- Torch/pickle deserialization of the embedding model is inferred from the sentence-transformers/InstructorEmbedding loader, not read in this repo.
- Third-party plugin services' own behaviour is out of scope; only how the platform calls them was reviewed.
- No reviewer-directed prompt injection found in the repo.
