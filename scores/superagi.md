# Defense-in-Depth Score: SuperAGI

**Repo:** https://github.com/transformeroptimus/superagi · **Commit:** `c3c1982e7bd6a11cfed53c5a193ea502f924b1b6` · **Reviewed:** 2026-10-04
**What it is:** Open-source autonomous agent platform with a web GUI, tool marketplace, workflows and scheduled agents.
**Category:** Agent Frameworks
**Scored configuration:** Shipped docker-compose.yaml with config_template.yaml defaults (ENV DEV) and GUI agent-creation defaults (God Mode, Goal Based Workflow, SearxSearch/Read File/Write File).
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents no · external communication yes

## Score: 1.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L1 | L1 | L0 | L0 | 0.15 | C2-SELFAPPROVE | **0.15** | High |
| C3 | Tool & action scoping | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | C7-RCELOAD | **0.00** | High |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C9 | Audit & traceability | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |
| C10 | Limits & kill switch | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |


As shipped, SuperAGI has essentially no safety boundary: agents default to 'God Mode' with no approval, every agent uses the operator's global email, GitHub and chat credentials, and every container start downloads and runs unpinned third-party tool code. The API's access control is also not locked down, model output in task workflows is not confined, and the file tools' path handling is not a strict boundary. A single prompt injection from a search result or email can exfiltrate secrets and take irreversible actions unattended.

## Critical gaps
- A hijacked agent in default God Mode can exfiltrate secrets and take irreversible actions (email, tweets, GitHub deletes) with no human involved. (ASI01, T6, LLM01; C5) — [superagi/agent/output_handler.py:55-58](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/agent/output_handler.py#L55-L58); [gui/pages/Content/Agents/AgentCreate.js:97-98](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/gui/pages/Content/Agents/AgentCreate.js#L97-L98); [superagi/tools/email/send_email.py:81-85](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/tools/email/send_email.py#L81-L85)
- Every container start downloads unpinned main-branch tool code from GitHub, pip-installs its requirements and imports it in-process with no integrity check. (ASI04, T17, LLM03; C7) — [superagi/tool_manager.py:56-57](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/tool_manager.py#L56-L57); [entrypoint_celery.sh:3-7](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/entrypoint_celery.sh#L3-L7); [superagi/helper/tool_helper.py:113-116](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/helper/tool_helper.py#L113-L116)
- Every agent uses global operator credentials with write access to multiple external systems; API access control is not locked down. (ASI03, T3, T9; C1) — [config_template.yaml:89](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/config_template.yaml#L89)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

SuperAGI runs every agent with one set of operator credentials read from a global config.yaml (email password, GitHub token, Slack, Jira, AWS), shared by every user and organisation. The API's authentication and authorization are not locked down.

- **S L0:** Tools use broad, long-lived operator credentials from the global config; there is no per-agent or per-user identity. — [config_template.yaml:79](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/config_template.yaml#L79); [config_template.yaml:89](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/config_template.yaml#L89) (verified)
  - *To reach the next level:* No dedicated, role-scoped identity per agent or tool.
- **C L0:** No authorization layer covers tools and endpoints; endpoint access control is not locked down. (verified)
  - *To reach the next level:* No authorization layer that every tool and endpoint passes through.
- **D L0:** The default install grants every agent the global operator credentials with no least-privilege profile; access control in the default ENV is not locked down. (verified)
  - *To reach the next level:* Default install should require authentication and least-privilege credentials.
- **B L0:** A hijacked agent can send email, push/delete GitHub files, post to Slack/Twitter and edit Jira with the operator's accounts. — [config_template.yaml:79](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/config_template.yaml#L79); [config_template.yaml:89](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/config_template.yaml#L89); [superagi/tools/email/send_email.py:81-85](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/tools/email/send_email.py#L81-L85) (verified)
  - *To reach the next level:* Credentials would need to be scoped to one system and mostly read-only.
- **Cap:** none
- **Notes:** Authentication on the API is not locked down.

### C2 Approval gates — 0.15 (high)

A human-approval mode (RESTRICTED) exists, but agents default to 'God Mode', where every tool call runs without approval; agents created through the public API are forced into God Mode. When RESTRICTED is chosen, the approver is shown only the tool's name, not its arguments, and the approval endpoint's access control is not locked down. Consequential actions such as sending email, deleting files or posting tweets are irreversible.

- **S L1:** Per-call approval exists in RESTRICTED mode, but the console shows only the tool name, so the approver cannot see recipients, paths or content. — [superagi/agent/output_handler.py:112-113](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/agent/output_handler.py#L112-L113); [gui/pages/Content/Agents/ActionConsole.js:15](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/gui/pages/Content/Agents/ActionConsole.js#L15) (verified)
  - *To reach the next level:* Approver must see the exact call (arguments, recipients, diffs).
- **C L1:** Only tools with permission_required are gated, only in RESTRICTED mode; API-created agents are forced to God Mode, and gate coverage of task-workflow paths is incomplete. — [superagi/agent/output_handler.py:112-113](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/agent/output_handler.py#L112-L113); [superagi/controllers/api/agent.py:68](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/controllers/api/agent.py#L68) (verified)
  - *To reach the next level:* Every tool path, API-created agents and task workflows must traverse the gate.
- **D L0:** The GUI default permission type is 'God Mode' (no approval). — [gui/pages/Content/Agents/AgentCreate.js:97-98](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/gui/pages/Content/Agents/AgentCreate.js#L97-L98); [gui/pages/Content/Agents/AgentCreate.js:97](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/gui/pages/Content/Agents/AgentCreate.js#L97) (verified)
  - *To reach the next level:* Approval should be on by default.
- **B L0:** Wrongly approved or ungated calls can send external email, tweet, delete GitHub files or local files, with no undo or preview. — [superagi/tools/email/send_email.py:81-85](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/tools/email/send_email.py#L81-L85); [superagi/tools/file/delete_file.py:60-61](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/tools/file/delete_file.py#L60-L61) (verified)
  - *To reach the next level:* Need reversibility or previews for common actions.
- **Cap:** C2-SELFAPPROVE — The approval endpoint's access control is not locked down, which lets a non-principal approve.

### C3 Tool & action scoping — 0.05 (high)

Tools pass model arguments straight through. File tools' path handling is not a strict boundary. The web scraper fetches any URL, including internal addresses. Pydantic schemas only check types. The default tool set offered in the GUI includes Write File.

- **S L0:** File-path handling is not a strict boundary, and the scraper fetches arbitrary URLs. — [superagi/helper/webpage_extractor.py:101](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/helper/webpage_extractor.py#L101) (verified)
  - *To reach the next level:* Path containment and URL/host allowlists in code.
- **C L0:** No built-in tool validates paths, URLs or recipients beyond pydantic type parsing. — [superagi/tools/base_tool.py:116](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/tools/base_tool.py#L116) (verified)
  - *To reach the next level:* At least some tools should validate arguments against bounds.
- **D L1:** Tools are chosen per agent; the GUI pre-selects SearxSearch, Read File and Write File, and any installed tool can be added. — [gui/pages/Content/Agents/AgentCreate.js:103](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/gui/pages/Content/Agents/AgentCreate.js#L103) (verified)
  - *To reach the next level:* Default tool set should be read-only, with write/exec explicitly enabled.
- **B L0:** A misused file tool reaches beyond the agent workspace, and the scraper reaches any host the server can. — [docker-compose.yaml:13-15](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/docker-compose.yaml#L13-L15) (verified)
  - *To reach the next level:* Tools should be scoped to the agent workspace.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

SuperAGI has no sandbox. Not every model-influenced execution path in task workflows is confined. The worker process holds every credential, runs as root in a container with no hardening, and has the application source mounted read-write from the host. Startup also runs downloaded tool code in-process.

- **S L0:** No isolation primitive exists around model-influenced execution, including the task-workflow paths. (verified)
  - *To reach the next level:* An OS-level or language-runtime boundary around model-influenced code.
- **C L0:** No execution path is sandboxed. — searched `rg -n 'cap_drop|no-new-privileges|seccomp|read_only|user:' docker-compose.yaml` in `docker-compose.yaml` → 0 hits (no container hardening in the shipped compose) (verified)
  - *To reach the next level:* At least the main exec path must be isolated.
- **D L0:** There is no sandbox to enable. — searched `rg -n 'cap_drop|no-new-privileges|seccomp|read_only|user:' docker-compose.yaml` in `docker-compose.yaml` → 0 hits (no container hardening in the shipped compose); searched `rg -n '^USER' Dockerfile DockerfileCelery` in `Dockerfile DockerfileCelery` → 0 hits (images run as root) (verified)
  - *To reach the next level:* A sandbox on by default.
- **B L0:** Code runs as root in the worker that holds all API keys and DB credentials, with the source tree mounted read-write from the host and unrestricted network. — [docker-compose.yaml:13-15](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/docker-compose.yaml#L13-L15); searched `rg -n '^USER' Dockerfile DockerfileCelery` in `Dockerfile DockerfileCelery` → 0 hits (images run as root); [config_template.yaml:79](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/config_template.yaml#L79) (verified)
  - *To reach the next level:* Workspace-only mount, no secrets in the environment, restricted egress.
- **Cap:** none
- **Notes:** Not every task-workflow execution path is confined.

### C5 Untrusted input blast radius — 0.00 (high)

Search results, scraped pages, emails, GitHub PRs and files enter the model's context with no marking, and tool results are stored and replayed with the 'system' role, giving injected text the same standing as the operator's instructions. Nothing limits a hijacked agent: in the default God Mode it can read secrets and send them out through email, Slack, Twitter or a scraped URL, and take irreversible actions, all unattended.

- **S L0:** No structural limit or detection on content from tools. — searched `rg -n -i 'untrusted|injection|sanitiz' superagi/agent` in `superagi/agent` → 0 hits (no provenance or untrusted-content handling in the agent loop); [superagi/agent/output_handler.py:55-58](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/agent/output_handler.py#L55-L58) (verified)
  - *To reach the next level:* Approval for egress/state change once untrusted content has been read.
- **C L0:** Tool results are stored with role 'system' and replayed into later prompts as such. — [superagi/agent/output_handler.py:55-58](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/agent/output_handler.py#L55-L58); [superagi/agent/agent_message_builder.py:52](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/agent/agent_message_builder.py#L52) (verified)
  - *To reach the next level:* Tool results must be distinguished from principal instructions.
- **D L0:** There is no control to turn on. — searched `rg -n -i 'untrusted|injection|sanitiz' superagi/agent` in `superagi/agent` → 0 hits (no provenance or untrusted-content handling in the agent loop) (verified)
  - *To reach the next level:* A provenance or taint control on by default.
- **B L0:** Default tools (search, read/write file) plus email, Slack, Twitter and arbitrary URL fetch allow exfiltration and irreversible actions with no human in God Mode. — [gui/pages/Content/Agents/AgentCreate.js:103](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/gui/pages/Content/Agents/AgentCreate.js#L103); [superagi/helper/webpage_extractor.py:101](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/helper/webpage_extractor.py#L101); [superagi/tools/email/send_email.py:81-85](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/tools/email/send_email.py#L81-L85); [gui/pages/Content/Agents/AgentCreate.js:97-98](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/gui/pages/Content/Agents/AgentCreate.js#L97-L98) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions should require human approval.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.10 (high)

When the OpenAI model is used, every model reply and raw tool result is written unvalidated into a Redis vector index shared by all agents and organisations, and later read back into a system-role prompt by the thinking tool. Retrieval is filtered by execution id, so this store mostly affects the same run. A more durable path exists through the file tools.

- **S L0:** The model writes anything into memory and it is re-injected as system-role context. — [superagi/agent/output_handler.py:87-92](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/agent/output_handler.py#L87-L92); [superagi/tools/thinking/tools.py:66-67](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/tools/thinking/tools.py#L66-L67) (verified)
  - *To reach the next level:* Memory entries should carry provenance and be presented as data.
- **C L0:** No memory or persisted-context path is validated. — [superagi/agent/output_handler.py:87-92](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/agent/output_handler.py#L87-L92) (verified)
  - *To reach the next level:* At least the main store should be controlled.
- **D L1:** One global index name is used for all tenants; isolation depends on a caller-supplied metadata filter in the one retrieval path. — [superagi/jobs/agent_executor.py:76](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/jobs/agent_executor.py#L76); [superagi/tools/thinking/tools.py:64-65](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/tools/thinking/tools.py#L64-L65) (verified)
  - *To reach the next level:* Per-user/session namespaces enforced in the store, not by an optional filter.
- **B L1:** Files written through the file tools can persist across that agent's runs and steer later tool use. — [config_template.yaml:38](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/config_template.yaml#L38) (verified)
  - *To reach the next level:* Poisoned context should only influence text output or gated actions.
- **Cap:** none

### C7 Third-party extensions — 0.00 (high)

Every time the backend or worker container starts, it downloads the latest main-branch zip of the TransformerOptimus/SuperAGI-Tools repository (and any user-added GitHub tool links), without pinning or integrity checks, pip-installs each tool's requirements as root, and imports the tool code into the server process. Any user can add a new GitHub tool link through the API. A compromise of that repository or any listed tool runs with all of SuperAGI's credentials.

- **S L0:** Unpinned main-branch archives are downloaded and executed automatically at every start. — [superagi/tool_manager.py:56-57](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/tool_manager.py#L56-L57); [superagi/tool_manager.py:109](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/tool_manager.py#L109); [entrypoint_celery.sh:3-7](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/entrypoint_celery.sh#L3-L7); searched `rg -n -i 'sha256|checksum|verify|signature' superagi/tool_manager.py superagi/helper/tool_helper.py` in `superagi/tool_manager.py superagi/helper/tool_helper.py` → 0 hits (no integrity check on downloaded tool code) (verified)
  - *To reach the next level:* Versions should at least be pinned.
- **C L0:** No extension type (marketplace or external tool) is verified. — [superagi/tool_manager.py:64-71](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/tool_manager.py#L64-L71); [superagi/helper/tool_helper.py:113-116](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/helper/tool_helper.py#L113-L116); searched `rg -n -i 'sha256|checksum|verify|signature' superagi/tool_manager.py superagi/helper/tool_helper.py` in `superagi/tool_manager.py superagi/helper/tool_helper.py` → 0 hits (no integrity check on downloaded tool code) (verified)
  - *To reach the next level:* Verification for at least one extension type.
- **D L0:** All marketplace tools are fetched and installed by default; an API call can add arbitrary GitHub tool repos to tools.json. — [superagi/tool_manager.py:137-139](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/tool_manager.py#L137-L139); [superagi/controllers/toolkit.py:216-222](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/controllers/toolkit.py#L216-L222) (verified)
  - *To reach the next level:* Nothing third-party enabled by default; explicit, informed install.
- **B L0:** Tool code is loaded in-process via exec_module, with all credentials and the root user. — [superagi/helper/tool_helper.py:113-116](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/helper/tool_helper.py#L113-L116); [install_tool_dependencies.sh:17-19](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/install_tool_dependencies.sh#L17-L19) (verified)
  - *To reach the next level:* Separate process with scrubbed environment.
- **Cap:** C7-RCELOAD — By default every container start downloads and executes unpinned remote tool code without consent.

### C8 Secrets & sensitive-data protection — 0.00 (high)

Credentials live in a plaintext config.yaml, and default secrets and stored tool keys are not locked down. Nothing is redacted: tool outputs are logged at info level. Analytics are only enabled when the operator provides IDs.

- **S L0:** Secret handling is not locked down; no redaction anywhere. — searched `rg -n -i 'redact|mask|SecretStr' superagi` in `superagi` → 0 hits (no redaction or secret-typing anywhere) (verified)
  - *To reach the next level:* Secrets from env/secret store, and masking on at least one path.
- **C L0:** No path (logs, model context, API responses, subprocess env) is protected. — [superagi/agent/tool_executor.py:64](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/agent/tool_executor.py#L64); searched `rg -n -i 'redact|mask|SecretStr' superagi` in `superagi` → 0 hits (no redaction or secret-typing anywhere) (verified)
  - *To reach the next level:* At least one path protected by redaction.
- **D L0:** Tool outputs are logged in full at the default info level. — [superagi/agent/tool_executor.py:64](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/agent/tool_executor.py#L64); [entrypoint_celery.sh:3-7](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/entrypoint_celery.sh#L3-L7) (verified)
  - *To reach the next level:* Payload logging should be off or redacted by default.
- **B L0:** Long-lived, high-privilege operator keys are reachable by every tool in-process. — [config_template.yaml:79](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/config_template.yaml#L79); [config_template.yaml:89](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/config_template.yaml#L89) (verified)
  - *To reach the next level:* Keys should be scoped and short-lived.
- **Cap:** none
- **Notes:** Mixpanel/GA in the GUI only initialise when env IDs are supplied (gui/pages/api/apiConfig.js:6-8, gui/pages/_app.js:116).

### C9 Audit & traceability — 0.38 (high)

Each step's raw model reply (which contains the tool name and arguments) and the tool's result are saved as timestamped rows in Postgres, and approval decisions are stored in their own table. There is no actor attribution beyond a single shared user, no tamper protection, and the worker process that executes model-controlled code holds the database credentials. Records are written after the tool runs, so a crash mid-call loses it.

- **S L2:** Assistant reply (tool call JSON with arguments) and tool result are stored per step with created_at timestamps. — [superagi/agent/output_handler.py:49-61](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/agent/output_handler.py#L49-L61); [superagi/models/base_model.py:27](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/models/base_model.py#L27) (verified)
  - *To reach the next level:* No actor attribution (requesting principal, approver identity) or correlation IDs.
- **C L1:** Main tool path recorded; some task-workflow execution paths and memory/config writes are not recorded as actions. — [superagi/agent/output_handler.py:49-61](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/agent/output_handler.py#L49-L61) (verified)
  - *To reach the next level:* All execution paths including task-queue paths and approvals with identity.
- **D L2:** On by default in Postgres outside the workspace, but the worker process holds DB credentials and can alter rows. (verified)
  - *To reach the next level:* Records written by a component the model can't control.
- **B L1:** Feed rows are committed only after the tool returns; failures are logged and the step is retried. — [superagi/agent/output_handler.py:49-61](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/agent/output_handler.py#L49-L61); [superagi/jobs/agent_executor.py:89-92](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/jobs/agent_executor.py#L89-L92) (verified)
  - *To reach the next level:* Records flushed per action before execution with errors surfaced.
- **Cap:** none

### C10 Limits & kill switch — 0.38 (high)

Each run has an iteration cap (25 by default in the GUI) enforced before every step, and runs older than one day are skipped. There is no token or cost cap: a Budget table exists but is never checked. Stopping a run only changes a status field that the next step checks, so an in-flight tool call finishes, and scheduled agents keep firing from the Celery beat scheduler.

- **S L2:** Iteration cap plus a coarse one-day age limit are enforced in code; no token/cost cap and stop is cooperative. — [superagi/jobs/agent_executor.py:145](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/jobs/agent_executor.py#L145); [superagi/jobs/agent_executor.py:47](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/jobs/agent_executor.py#L47); searched `rg -n 'Budget' superagi/jobs superagi/agent superagi/worker.py` in `superagi/jobs superagi/agent superagi/worker.py` → 0 hits (Budget model exists but is never enforced in the execution path) (verified)
  - *To reach the next level:* Token/cost caps and rate limits on side-effecting tools.
- **C L1:** Limits apply to the top-level loop; tool calls have no timeouts except the scraper's 10s HTTP timeout. — [superagi/jobs/agent_executor.py:145](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/jobs/agent_executor.py#L145); [superagi/helper/webpage_extractor.py:101](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/helper/webpage_extractor.py#L101) (verified)
  - *To reach the next level:* Tool timeouts across all tools.
- **D L2:** Sensible GUI default of 25 iterations; operator-configurable, the model cannot change it. — [gui/pages/Content/Agents/AgentCreate.js:48](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/gui/pages/Content/Agents/AgentCreate.js#L48); [superagi/jobs/agent_executor.py:145](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/jobs/agent_executor.py#L145) (verified)
  - *To reach the next level:* The limit also bounds delegation/scheduled runs and has a hard ceiling.
- **B L1:** Ceilings are large (a day of wall clock, unlimited spend); stop leaves in-flight work running and scheduled runs continue. — [superagi/controllers/agent_execution.py:348](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/controllers/agent_execution.py#L348); [superagi/jobs/agent_executor.py:53-56](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/jobs/agent_executor.py#L53-L56); [superagi/worker.py:32-35](https://github.com/transformeroptimus/superagi/blob/c3c1982e7bd6a11cfed53c5a193ea502f924b1b6/superagi/worker.py#L32-L35); searched `rg -n 'revoke|terminate|SIGTERM|kill' superagi/jobs superagi/worker.py superagi/controllers/agent_execution.py` in `superagi/jobs superagi/worker.py superagi/controllers/agent_execution.py` → 0 hits (no task revocation or process kill on stop) (verified)
  - *To reach the next level:* Tight time/cost ceilings and cancellation of pending calls.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Search/scrape/email/GitHub tools; tool results replayed as system role (superagi/agent/output_handler.py:55-58) · [B] sensitive data/systems: Global config.yaml operator credentials used by every agent (config_template.yaml:79) · [C] state change / egress: Email/Slack/Twitter/GitHub tools and arbitrary URL fetch (superagi/helper/webpage_extractor.py:101), ungated in God Mode · Same default session? Yes

## Highest-impact improvements
1. Default agents to RESTRICTED mode and show full tool arguments in the approval console. — C2 D L0→L3, +0.150 before caps (Playbook 5)
2. Harden how task workflows parse model output. — C4 B L0→L2, +0.100 before caps (Playbook 3)
3. Contain file tools to the agent's workspace; allowlist scraper hosts and block internal addresses. — C3 S L0→L3, +0.225 before caps (Playbook 3)
4. Pin marketplace tools to commit hashes with digest checks and stop auto-downloading at every start. — C7 S L0→L3, +0.225 before caps (Playbook 3)
5. Harden default authentication and authorization on the API. — C1 D L0→L2, +0.100 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Third-party marketplace tool code (TransformerOptimus/SuperAGI-Tools) is fetched at runtime and was not examined.
- The GUI was reviewed only for agent-creation defaults, the approval console and analytics initialisation.
- No text aimed at AI reviewers was found in the repository.
