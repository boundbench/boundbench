# Defense-in-Depth Score: OpenAGI (pyopenagi)

**Repo:** https://github.com/agiresearch/openagi · **Commit:** `26c119e55b8d6ad4bae96cd4e5901c1567a82aaa` (0.0.11) · **Reviewed:** 2026-10-04
**What it is:** Python package for building tool-using agents that run on the AIOS kernel, with a ReAct agent base class, bundled API tools, and an agent hub client.
**Category:** Agent Frameworks
**Scored configuration:** pyopenagi 0.0.11 defaults: AgentFactory.run_agent with a ReactAgent subclass, console logging, tools from the agent's config.json, LLM calls via the AIOS queue.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents no · external communication no

## Score: 2.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C2 | Approval gates | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C3 | Tool & action scoping | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | C7-RCELOAD | **0.00** | High |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C9 | Audit & traceability | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |

Controls where a risk surface exists: 1.08 / 9.0 (12%); 1 criterion scored SA (surface absent).

OpenAGI runs whatever tool calls the model returns with no approval, no sandbox, and no budget, using long-lived API keys from the environment. The dominant risk is its agent hub: activating an agent that is not installed locally silently downloads its code from a hosted endpoint, pip-installs its requirements and imports it, with no pin or integrity check. The only argument check, a path rewrite, is not a complete boundary. The package is no longer the maintained SDK (the README points to Cerebrum), so treat it as a research prototype.

## Critical gaps
- Activating an agent auto-downloads unpinned, unverified Python code and requirements from a hosted hub, pip-installs them and imports the code into the agent process. (ASI04, T17, LLM03; C7) — [pyopenagi/agents/agent_factory.py:57-61](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/agent_factory.py#L57-L61); [pyopenagi/agents/interact.py:41](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/interact.py#L41); [pyopenagi/agents/agent_factory.py:43](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/agent_factory.py#L43); [pyopenagi/agents/interact.py:217-223](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/interact.py#L217-L223)
- Package installs and eval() run directly on the host as the operator with the full environment and every API key; no isolation exists. (ASI05, T11; C4) — [pyopenagi/agents/interact.py:217-223](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/interact.py#L217-L223); [pyopenagi/tools/travel_planner/google_distance_matrix.py:35](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/tools/travel_planner/google_distance_matrix.py#L35)

## Criterion details

### C1 Identity & least privilege — 0.05 (high)

OpenAGI has no identity or authorization layer of its own. Every bundled tool reads a long-lived API key (a shared RapidAPI key, a Hugging Face token, a Bing key) straight from the process environment and calls the vendor API with it. Nothing narrows these keys per tool or per request, and the same environment is inherited by the pip subprocesses and by any agent code downloaded from the hub. A hijacked agent therefore acts with whatever the operator's keys allow.

- **S L0:** Tools use the operator's own long-lived API keys taken from os.environ; no dedicated or scoped identity exists. — [pyopenagi/utils/utils.py:41-43](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/utils/utils.py#L41-L43); [pyopenagi/tools/trip_advisor/airport_search.py:14](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/tools/trip_advisor/airport_search.py#L14) (verified)
  - *To reach the next level:* No dedicated agent identity or per-tool scoped credentials.
- **C L0:** Each tool constructs its own client headers from environment keys; there is no authorization check in the executor. — searched `rg -n -F get_from_env(` in `pyopenagi/tools` → 25 hits (every bundled API tool reads its key straight from the process environment); [pyopenagi/agents/react_agent.py:86-90](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/react_agent.py#L86-L90) (verified)
  - *To reach the next level:* No shared authorization layer on the tool path.
- **D L0:** The default install hands every tool the full environment; least privilege would need manual hardening by the developer. — [pyopenagi/utils/utils.py:41-43](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/utils/utils.py#L41-L43) (verified)
  - *To reach the next level:* No narrower default role or credential set.
- **B L1:** Keys reach several third-party accounts (RapidAPI account key shared by ~15 tools, Hugging Face token, Bing); a Hugging Face token can be write-scoped. — [pyopenagi/tools/trip_advisor/airport_search.py:14](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/tools/trip_advisor/airport_search.py#L14); [pyopenagi/tools/impira/doc_question_answering.py:16](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/tools/impira/doc_question_answering.py#L16) (verified)
  - *To reach the next level:* Credentials are not limited to a single system or to read-only scope.
- **Cap:** none

### C2 Approval gates — 0.05 (high)

There is no approval step anywhere in the framework. The ReAct executor takes whatever tool calls the model returns and runs them immediately, including tools that write files to disk or send local files to a third-party API. Developers who register their own tools (email, payments, shell) get the same ungated execution. The bundled tools are mostly read-only lookups, which limits but does not remove the damage.

- **S L0:** Tool calls returned by the model are executed directly with no human or policy check. — [pyopenagi/agents/react_agent.py:86-90](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/react_agent.py#L86-L90); searched `rg -n -i 'approv|confirm|human_in|input\(' --glob '!*.txt'` in `pyopenagi` → 0 hits (no approval, confirmation, or human-input primitive anywhere in the package) (verified)
  - *To reach the next level:* No per-call human approval of any kind.
- **C L0:** No gate exists, so every tool path, including the file-writing tools, is ungated. — searched `rg -n -i 'approv|confirm|human_in|input\(' --glob '!*.txt'` in `pyopenagi` → 0 hits (no approval, confirmation, or human-input primitive anywhere in the package) (verified)
  - *To reach the next level:* No gate for any tool path.
- **D L0:** Approval is not available even as an option. — searched `rg -n -i 'approv|confirm|human_in|input\(' --glob '!*.txt'` in `pyopenagi` → 0 hits (no approval, confirmation, or human-input primitive anywhere in the package) (verified)
  - *To reach the next level:* No approval primitive to enable by default.
- **B L1:** Bundled side effects are file writes and paid API calls, which are not reversible; framework users can register far more powerful tools that run equally ungated. — [pyopenagi/tools/suno/text_to_speech.py:24](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/tools/suno/text_to_speech.py#L24) (verified)
  - *To reach the next level:* No checkpoint or undo for file writes and no bound on consequential calls.
- **Cap:** none

### C3 Tool & action scoping — 0.38 (high)

Tools are narrow by construction: each one calls a single fixed vendor endpoint, and an agent only gets the tools its config.json lists. The one shared argument check rewrites any parameter whose name contains 'path' into an output folder, but its containment is not a complete boundary. Other arguments (queries, durations) are passed through without bounds. The model cannot add tools at runtime, but nothing stops an agent config from enabling write tools.

- **S L2:** Fixed-host tools with JSON schemas plus a path-rewrite helper whose containment is not a complete boundary. — [pyopenagi/tools/imdb/top_movies.py:12](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/tools/imdb/top_movies.py#L12) (verified)
  - *To reach the next level:* Path containment is not robust and other arguments have no numeric or allowlist bounds.
- **C L1:** check_path runs on every tool call in the ReactAgent executor, but it only covers parameters named like 'path' (a few file tools); query, duration and other arguments of most tools are unvalidated, and custom agents that override run() skip it. — [pyopenagi/agents/react_agent.py:171](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/react_agent.py#L171) (verified)
  - *To reach the next level:* Most built-in tools have no argument validation, and agents outside ReactAgent bypass check_path.
- **D L2:** Tools are enabled per agent from config.json and the model can only call tools in that list, but there is no read-only default and example agents enable file-writing tools. — [pyopenagi/agents/base_agent.py:157-159](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/base_agent.py#L157-L159); [pyopenagi/agents/react_agent.py:86-90](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/react_agent.py#L86-L90) (verified)
  - *To reach the next level:* No read-only default tool set; write tools are enabled by plain config.
- **B L1:** File-writing tools are not reliably contained and doc_question_answering can read any local file and upload it to Hugging Face. — [pyopenagi/tools/suno/text_to_speech.py:24](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/tools/suno/text_to_speech.py#L24); [pyopenagi/tools/impira/doc_question_answering.py:22](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/tools/impira/doc_question_answering.py#L22) (verified)
  - *To reach the next level:* Writes are not confined to a workspace and reads are not confined at all.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

The framework gives the model no code or shell tool, but it does execute third-party and data-supplied code on the host with no isolation. Activating an agent runs 'pip install -r' on that agent's requirements (package install scripts run as the user with the full environment), and the travel-planner distance tool calls Python eval() on values from a data file. Nothing in the repository provides a sandbox, container, or restricted user, so anything that runs has the operator's full access and API keys.

- **S L0:** pip install and eval() run as the same user on the host; no isolation primitive exists. — [pyopenagi/agents/interact.py:217-223](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/interact.py#L217-L223); [pyopenagi/tools/travel_planner/google_distance_matrix.py:35](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/tools/travel_planner/google_distance_matrix.py#L35); searched `rg -n -i 'sandbox|docker|container|seccomp'` in `pyopenagi` → 3 hits (all 3 hits are game-recommendation task text in data/agent_tasks; no isolation code) (verified)
  - *To reach the next level:* No OS-level separation (container, low-privilege user) for code that runs.
- **C L0:** No execution path is sandboxed. — searched `rg -n -i 'sandbox|docker|container|seccomp'` in `pyopenagi` → 3 hits (all 3 hits are game-recommendation task text in data/agent_tasks; no isolation code) (verified)
  - *To reach the next level:* No execution path goes through an isolation boundary.
- **D L0:** There is no sandbox to turn on. — searched `rg -n -i 'sandbox|docker|container|seccomp'` in `pyopenagi` → 3 hits (all 3 hits are game-recommendation task text in data/agent_tasks; no isolation code) (verified)
  - *To reach the next level:* No sandbox available or on by default.
- **B L0:** Install scripts and eval'd code run with the host filesystem, full network, and every API key in os.environ. — [pyopenagi/agents/interact.py:217-223](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/interact.py#L217-L223); searched `rg -n env=` in `pyopenagi` → 0 hits (no subprocess is given a scrubbed environment; pip inherits the full os.environ) (verified)
  - *To reach the next level:* Executed code is not separated from the host environment or credentials.
- **Cap:** none

### C5 Untrusted input blast radius — 0.05 (high)

Results from web search, Wikipedia, arXiv and travel APIs are pasted straight into the conversation as assistant messages, with the same standing as the agent's own reasoning. Nothing marks them as untrusted or restricts what the agent can do after reading them. A hijacked agent can write files and push local files to a third-party API without anyone approving it. The bundled tools offer no attacker-chosen URL, which limits direct exfiltration, but framework users' own tools get no protection.

- **S L0:** No structural limit on a hijacked agent; tool output simply joins the message list. — [pyopenagi/agents/react_agent.py:177-181](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/react_agent.py#L177-L181) (verified)
  - *To reach the next level:* No approval or capability restriction once untrusted content is read.
- **C L0:** Untrusted tool results enter context as role 'assistant', indistinguishable from the agent's own turns. — [pyopenagi/agents/react_agent.py:177-181](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/react_agent.py#L177-L181) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished from principal input.
- **D L0:** No defense exists to enable. — searched `rg -n -i 'approv|confirm|human_in|input\(' --glob '!*.txt'` in `pyopenagi` → 0 hits (no approval, confirmation, or human-input primitive anywhere in the package) (verified)
  - *To reach the next level:* No untrusted-input control on by default.
- **B L1:** Unattended irreversible actions are possible (file writes, local file upload to Hugging Face), but bundled tools hit only fixed vendor hosts, so there is no attacker-observable exfiltration channel by default. — [pyopenagi/tools/suno/text_to_speech.py:24](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/tools/suno/text_to_speech.py#L24); [pyopenagi/tools/impira/doc_question_answering.py:22](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/tools/impira/doc_question_answering.py#L22) (verified)
  - *To reach the next level:* Irreversible actions are not behind human approval.
- **Cap:** none

### C6 Memory, context & configuration integrity — 1.00 (high)

OpenAGI keeps no memory the model can write to. Conversation messages live only in the agent object for one run, logs are written but never read back, and there is no .env loading or auto-loaded instruction file. The RAG example builds a local vector store, but only from an essay shipped with the agent, not from anything the model or a user supplies. Agent packages downloaded from the hub do persist on disk and are re-imported later; that risk is scored under third-party extensions.

- **Structural absence:** searched `rg -n -S 'dotenv|chroma|persist|save_memory|remember|vector' --glob '!*.txt'` in `pyopenagi` → 18 hits (all 18 hits are in the RAG example agent, which indexes a bundled essay (data/paul_graham) once; no model- or user-writable store, no load_dotenv); searched `rg -n -S load_dotenv` in `pyopenagi` → 0 hits (python-dotenv is listed in requirements but never called)

### C7 Third-party extensions — 0.00 (high)

This is the most serious problem. When an agent name is activated and its folder is not present locally, the framework downloads that agent's Python code, config and requirements from a hosted hub (openagi-beta.vercel.app), writes them into the package, pip-installs the requirements, and imports the code, all without asking, without a version pin, and without any hash or signature check. Anyone who can publish or alter an agent on that hub, or who controls that endpoint, gets code execution in the operator's process with every API key in its environment.

- **S L0:** Remote agent code and requirements are fetched by name only (no version) and executed without verification. — [pyopenagi/agents/interact.py:41](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/interact.py#L41); [pyopenagi/agents/interact.py:176-180](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/interact.py#L176-L180); [pyopenagi/agents/agent_factory.py:43](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/agent_factory.py#L43); searched `rg -n 'sha256|hashlib'` in `pyopenagi` → 0 hits (no integrity check on downloaded agent code or requirements) (verified)
  - *To reach the next level:* No version pinning of downloaded agents or requirements.
- **C L0:** Neither downloaded agent code nor its pip requirements are verified. — searched `rg -n 'sha256|hashlib'` in `pyopenagi` → 0 hits (no integrity check on downloaded agent code or requirements); [pyopenagi/agents/interact.py:217-223](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/interact.py#L217-L223) (verified)
  - *To reach the next level:* No extension type is verified.
- **D L0:** activate_agent downloads and installs automatically whenever the agent folder is missing or a requirement is not installed. — [pyopenagi/agents/agent_factory.py:57-61](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/agent_factory.py#L57-L61) (verified)
  - *To reach the next level:* No explicit consent step before download and install.
- **B L0:** Downloaded agent code is imported into the agent's own process and pip runs with the full environment. — [pyopenagi/agents/agent_factory.py:43](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/agent_factory.py#L43); [pyopenagi/agents/interact.py:217-223](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/interact.py#L217-L223) (verified)
  - *To reach the next level:* Extensions run in-process with all credentials.
- **Cap:** C7-RCELOAD — AgentFactory.activate_agent downloads, pip-installs and imports remote agent code by default with no consent or verification.

### C8 Secrets & sensitive-data protection — 0.05 (high)

API keys come from environment variables and are never placed into the model prompt, which is the main thing done right. There is no masking or redaction anywhere, though: step results and tool parameters are printed to the console or appended to log files under the working directory, and the full environment, keys included, is inherited by pip subprocesses and by any downloaded agent code. There is no telemetry.

- **S L0:** Secrets are read from env vars but nothing masks them on any path. — [pyopenagi/utils/utils.py:41-43](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/utils/utils.py#L41-L43); searched `rg -n -i 'redact|mask|secretstr' --glob '!*.txt'` in `pyopenagi` → 0 hits (no redaction or masking helpers) (verified)
  - *To reach the next level:* No masking or redaction on even one path.
- **C L0:** No path (logs, subprocess environment, error messages) is protected. — searched `rg -n -i 'redact|mask|secretstr' --glob '!*.txt'` in `pyopenagi` → 0 hits (no redaction or masking helpers); searched `rg -n env=` in `pyopenagi` → 0 hits (no subprocess is given a scrubbed environment; pip inherits the full os.environ) (verified)
  - *To reach the next level:* No output path is protected.
- **D L1:** No telemetry ships; step output is logged unredacted to console by default and to cwd/logs in file mode. — [pyopenagi/utils/logger.py:19-20](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/utils/logger.py#L19-L20); [pyopenagi/utils/logger.py:66](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/utils/logger.py#L66) (verified)
  - *To reach the next level:* No redaction exists to keep on by default.
- **B L0:** Long-lived account keys are reachable by every subprocess and by downloaded agent code running in-process. — [pyopenagi/agents/interact.py:217-223](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/interact.py#L217-L223); [pyopenagi/agents/agent_factory.py:43](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/agent_factory.py#L43) (verified)
  - *To reach the next level:* Keys are not scoped or short-lived and are exposed to every subprocess.
- **Cap:** none

### C9 Audit & traceability — 0.25 (high)

The ReAct agent prints each step's result, which includes a sentence naming the tool called and its parameters, to the console by default, or appends it to a text file under ./logs in file mode. This is free text, not a structured record, has no actor or approval fields, and is written only after the step has run. Agents that override the run loop, or code using CallCore, record nothing about tool use.

- **S L1:** Unstructured per-step log lines that embed the tool name and parameters. — [pyopenagi/agents/react_agent.py:196](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/react_agent.py#L196); [pyopenagi/agents/react_agent.py:91](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/react_agent.py#L91) (verified)
  - *To reach the next level:* No structured record of each tool call with arguments, status and timestamps.
- **C L1:** Only the ReactAgent step loop logs tool use; custom run() implementations and CallCore do not. — [pyopenagi/agents/react_agent.py:196](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/react_agent.py#L196) (verified)
  - *To reach the next level:* Not every tool path is recorded.
- **D L1:** Logging is on by default but goes to the console; file mode writes under the working directory where the process can edit it. — [pyopenagi/utils/logger.py:19-20](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/utils/logger.py#L19-L20); [pyopenagi/utils/logger.py:66](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/utils/logger.py#L66) (verified)
  - *To reach the next level:* Records are not stored outside the agent's writable workspace.
- **B L1:** Lines are written after each step completes; console output is not persisted and write failures are not handled. — [pyopenagi/agents/react_agent.py:196](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/react_agent.py#L196); [pyopenagi/utils/logger.py:33](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/utils/logger.py#L33) (verified)
  - *To reach the next level:* Records are not flushed durably per action with surfaced errors.
- **Cap:** none

### C10 Limits & kill switch — 0.25 (high)

Limits are minimal. Planning and tool-call retries are capped at three, and manual workflows have a fixed number of steps, but in automatic mode the model writes the plan and so decides how many steps run. There is no wall-clock limit, token or cost budget, or rate limit in the framework, and the request loop re-queues until the external AIOS scheduler reports done. There is no stop or cancel mechanism; the terminate signal is commented out.

- **S L1:** Only retry counters (3) exist; no wall-clock, token or cost cap and no halt. — [pyopenagi/agents/react_agent.py:25](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/react_agent.py#L25); [pyopenagi/agents/agent_factory.py:24](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/agent_factory.py#L24) (verified)
  - *To reach the next level:* No wall-clock, token or cost limit enforced alongside the iteration cap.
- **C L1:** The retry caps apply only to the top-level ReactAgent loop; tools such as the transcriber sleep for a model-chosen duration with no timeout. — [pyopenagi/agents/base_agent.py:97](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/base_agent.py#L97); [pyopenagi/tools/transcriber/transcriber.py:30](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/tools/transcriber/transcriber.py#L30) (verified)
  - *To reach the next level:* No per-tool timeouts.
- **D L1:** Default retry limits exist but the model sets the number of steps in automatic mode. — [pyopenagi/agents/react_agent.py:136](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/react_agent.py#L136); [pyopenagi/agents/react_agent.py:25](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/react_agent.py#L25) (verified)
  - *To reach the next level:* The model can choose how many steps run.
- **B L1:** Runs end when the plan is exhausted, but there is no way to stop a running agent and the request loop has no exit other than the scheduler marking it done. — [pyopenagi/agents/base_agent.py:203](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/base_agent.py#L203); [pyopenagi/agents/agent_factory.py:24](https://github.com/agiresearch/openagi/blob/26c119e55b8d6ad4bae96cd4e5901c1567a82aaa/pyopenagi/agents/agent_factory.py#L24) (verified)
  - *To reach the next level:* No working stop that ends the loop.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web/Wikipedia/arXiv/travel API results appended as assistant messages (pyopenagi/agents/react_agent.py:177) · [B] sensitive data/systems: local files readable via doc_question_answering (pyopenagi/tools/impira/doc_question_answering.py:22) and env API keys (pyopenagi/utils/utils.py:42) · [C] state change / egress: file writes (pyopenagi/tools/suno/text_to_speech.py:24) and uploads to Hugging Face (pyopenagi/tools/impira/doc_question_answering.py:31) · Same default session? Yes

## Highest-impact improvements
1. Stop auto-downloading agents in activate_agent; require an explicit install command that pins a version and verifies a published hash before writing or importing code. — C7 D L0→L3, +0.150 before caps (Playbook 3)
2. Harden path containment in check_path. — C3 S L2→L3, +0.075 before caps (Playbook 3)
3. Add an optional-by-default-on approval callback in ReactAgent.call_tools that shows the exact tool name and parameters before running write or upload tools. — C2 S L0→L3, +0.225 before caps (Playbook 5)
4. Add a max_steps cap on automatic workflows plus a wall-clock limit and a cancel flag checked between steps. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
5. Run pip installs with a scrubbed environment (no API keys) so install scripts cannot read credentials. — C8 C L0→L1, +0.075 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of commit 26c119e only; nothing was executed, installed, or probed.
- Framework scored by its defaults: absent primitives score L0 even where a developer could add them.
- The LLM request path, scheduler, time limits and model-provider handling live in the external AIOS kernel (aios.hooks.stores._global), which was not examined.
- The hosted hub endpoint (openagi-beta.vercel.app) was not probed; whether it vets uploads is unknown and was not credited.
- Several example agents (e.g. rag_agent) import modules that do not exist at this commit and appear non-functional; they were read but not relied on for defaults.
- No reviewer-directed prompt injection found in the repo.
