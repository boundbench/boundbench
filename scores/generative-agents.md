# Defense-in-Depth Score: Generative Agents (Smallville)

**Repo:** https://github.com/joonspk-research/generative_agents · **Commit:** `fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4` · **Reviewed:** 2026-10-04
**What it is:** Research code for 'Generative Agents: Interactive Simulacra of Human Behavior': an LLM-driven multi-agent town simulation with a Django visual frontend.
**Category:** Agent Frameworks
**Scored configuration:** README setup: reverie.py interactive CLI forking base_the_ville_isabella_maria_klaus, with the Django frontend on localhost:8000 and a user-created utils.py (debug = True).
**Agent surface (default):** code execution no · filesystem write yes · network egress no · external credentials yes · persistent memory yes · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 4.4 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L2 | L3 | 0.55 | — | **0.55** | High |
| C2 | Approval gates | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C3 | Tool & action scoping | L3 | L2 | L4 | L3 | 0.72 | — | **0.72** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | Medium |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L2 | 0.15 | — | **0.15** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L0 | L1 | L1 | L1 | 0.17 | — | **0.17** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C10 | Limits & kill switch | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |

Controls where a risk surface exists: 2.40 / 8.0 (30%); 2 criteria scored SA (surface absent).

Generative Agents is a research simulation, not a tool-using agent: its LLM-driven characters can only walk around a fixed map, talk to each other and remember things, so most of its safety comes from having almost nothing dangerous to do. It ships no controls of its own. The main risk is in the local web frontend, whose handling of model-written text and of requests is not hardened. Memory is written by the model without checks and carries into every forked run.

## Critical gaps
- None: no criterion is capped and no blast radius is at the worst level.

## Criterion details

### C1 Identity & least privilege — 0.55 (high)

The simulation holds one credential: the operator's OpenAI API key, read from a hand-written utils.py and used only for model and embedding calls. The model has no tools, so it cannot use that key or any other operator credential to act on outside systems. Nothing narrows the key itself (it is a long-lived account key), and request handling on the local Django server that the simulation talks to is not locked down. A hijack therefore reaches the OpenAI quota and the local simulation files, not the operator's wider accounts.

- **S L2:** A single static credential (OpenAI key) is set globally for the SDK and used only for model calls; no other identity or ambient credential is loaded. — [reverie/backend_server/persona/prompt_template/gpt_structure.py:14](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/prompt_template/gpt_structure.py#L14); [README.md:18](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/README.md#L18) (verified)
  - *To reach the next level:* No per-capability or short-lived credential; the same long-lived account key serves every call.
- **C L2:** Every backend path uses the same module-level key and no subprocess or tool receives it, but access control on the Django endpoints that read and write simulation state is not locked down. — searched `rg -n -S --type py 'subprocess|os\.system|os\.popen|\beval\(|\bexec\('` in `reverie environment/frontend_server` → 0 hits (No shell, subprocess, eval or exec anywhere in the Python sources (backend or Django frontend).) (verified)
  - *To reach the next level:* Harden the local HTTP endpoints.
- **D L2:** Default is a reasonable single-purpose key; widening is a matter of what key the operator pastes into utils.py. — [README.md:18](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/README.md#L18) (verified)
  - *To reach the next level:* No read-only default or enforced scoping of the key; the README asks for a full account key with no scope guidance.
- **B L3:** A hijacked agent can only spend model quota and change local simulation files; it holds no write credential for any external system. — [reverie/backend_server/persona/prompt_template/gpt_structure.py:14](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/prompt_template/gpt_structure.py#L14); searched `rg -n -S --type py 'requests\.|urllib|smtplib|webdriver\.|socket\.|http\.client'` in `reverie environment/frontend_server` → 0 hits (No HTTP client, email, socket or browser-automation call besides the OpenAI SDK; selenium is imported but never used.) (verified)
  - *To reach the next level:* The key is long-lived and not revocable per run or per task.
- **Cap:** none

### C2 Approval gates — 1.00 (high)

The agents in this project take no consequential actions. Their model outputs only pick a destination on a fixed game map, an emoji, a short action description, chat lines and memory entries, all written to the simulation's own JSON files. There is no shell, HTTP client, email or other side-effecting tool for an approval gate to guard. The operator commands that delete data (exit) are typed by the human, not chosen by the model.

- **Structural absence:** searched `rg -n -S --type py 'subprocess|os\.system|os\.popen|\beval\(|\bexec\('` in `reverie environment/frontend_server` → 0 hits (No shell, subprocess, eval or exec anywhere in the Python sources (backend or Django frontend).); searched `rg -n -S --type py 'requests\.|urllib|smtplib|webdriver\.|socket\.|http\.client'` in `reverie environment/frontend_server` → 0 hits (No HTTP client, email, socket or browser-automation call besides the OpenAI SDK; selenium is imported but never used.)

### C3 Tool & action scoping — 0.72 (high)

The model's only real 'action' is choosing where a character walks, and that choice is checked against the map's fixed set of addresses before any path is computed. That is a narrow, allow-listed action space. The weak spot is how the text the model produces (chat lines and action descriptions) is handled in the browser, which is not sanitized. There are no tools to enable or disable and the model cannot add any.

- **S L3:** Movement targets are validated in code against the closed set of map addresses (maze.address_tiles). — [reverie/backend_server/persona/cognitive_modules/execute.py:91-94](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/cognitive_modules/execute.py#L91-L94) (verified)
  - *To reach the next level:* Model-written text fields are not validated, so not every output is covered.
- **C L2:** Movement is validated, but chat and description strings flow unchecked from the model to the page. — [reverie/backend_server/reverie.py:386-387](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/reverie.py#L386-L387) (verified)
  - *To reach the next level:* Chat and description outputs bypass any validation.
- **D L4:** There are no write, exec or network tools at all, and nothing lets the model load new tools. — searched `rg -n -S --type py 'subprocess|os\.system|os\.popen|\beval\(|\bexec\('` in `reverie environment/frontend_server` → 0 hits (No shell, subprocess, eval or exec anywhere in the Python sources (backend or Django frontend).); searched `rg -n -S --type py 'requests\.|urllib|smtplib|webdriver\.|socket\.|http\.client'` in `reverie environment/frontend_server` → 0 hits (No HTTP client, email, socket or browser-automation call besides the OpenAI SDK; selenium is imported but never used.); searched `rg -n -S --type py 'pickle\.load|torch\.load|trust_remote_code|importlib|__import__|pip install|entry_points'` in `reverie environment/frontend_server` → 0 hits (No dynamic code loading, model-file deserialization, or package installation.) (verified)
- **B L3:** A misused action can move a sprite or write text into the simulation files; the rendering of that text in the frontend is the only path with reach beyond the game. — [reverie/backend_server/persona/cognitive_modules/execute.py:91-94](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/cognitive_modules/execute.py#L91-L94) (verified)
  - *To reach the next level:* Text output is not bounded to inert content.
- **Cap:** none

### C4 Code-execution isolation — 0.05 (medium)

The project never runs model-generated code on purpose: there is no shell, eval or code tool. The report rates this criterion on how the simulator's browser frontend handles model-written text, which is not sanitized or isolated. The browser's own sandbox is the only boundary, and nothing in the project adds isolation.

- **S L0:** Model-produced text reaches the simulator page with no sanitization or isolation of its own. (verified)
  - *To reach the next level:* No sanitizer or isolation is applied to model-written content in the frontend.
- **C L0:** The single relevant path (browser rendering of model text) has no isolation. (verified)
  - *To reach the next level:* No path is isolated.
- **D L0:** No isolation exists to be on by default. (verified)
  - *To reach the next level:* No isolation is on by default.
- **B L1:** Impact is bounded to the browser and the local simulation server; the OpenAI key is not exposed to the page. (inferred)
  - *To reach the next level:* Local server write paths and open browser egress remain reachable.
- **Cap:** none

### C5 Untrusted input blast radius — 0.05 (medium)

The agents read content their operator did not write: other agents' chat lines, memories carried over in forked simulation folders, and history CSVs loaded on request. All of it goes into prompts with the same standing as the system's own instructions, and nothing structurally limits what a manipulated agent can then do. In practice the damage is small because agents have no tools, no access to secrets and no outbound channel from the backend. The exception is the browser frontend, where chat and description text is not sanitized.

- **S L0:** Nothing distinguishes or limits untrusted content; the only 'safety' check is an LLM score on anthropomorphization in interview mode. — [reverie/backend_server/persona/cognitive_modules/converse.py:136-146](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/cognitive_modules/converse.py#L136-L146); [reverie/backend_server/persona/cognitive_modules/converse.py:267](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/cognitive_modules/converse.py#L267) (verified)
  - *To reach the next level:* No structural limit applies once untrusted content is read.
- **C L0:** Other agents' utterances, forked memories and loaded history all enter prompts undifferentiated. — [reverie/backend_server/persona/cognitive_modules/converse.py:136-146](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/cognitive_modules/converse.py#L136-L146); [reverie/backend_server/reverie.py:56-58](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/reverie.py#L56-L58); [reverie/backend_server/reverie.py:579-591](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/reverie.py#L579-L591) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished from principal input.
- **D L0:** No control exists to be enabled. — [reverie/backend_server/persona/cognitive_modules/converse.py:136-146](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/cognitive_modules/converse.py#L136-L146) (verified)
  - *To reach the next level:* No untrusted-input control ships on.
- **B L1:** A hijacked agent cannot reach secrets or external systems from the backend, but the frontend's handling of its text gives an unattended outbound channel; leaked data is low-sensitivity simulation state. (inferred)
  - *To reach the next level:* An unattended exfiltration channel through the frontend remains.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.15 (high)

Each agent keeps a long-term memory of events, chats and model-written reflections that is saved to JSON and reloaded whenever a simulation is forked. Anything the model writes, including text from other agents, is stored without validation and later retrieved into prompts as trusted context. Each agent's memory lives in its own folder and every fork is a copy, so earlier simulation states remain intact, but there is no review, provenance tag or expiry enforcement on what gets remembered. A poisoned memory persists across the operator's sessions and can keep steering agents' output, including the chat shown in the frontend.

- **S L0:** The model writes reflections and chats straight into persistent associative memory, which is reloaded and retrieved into prompts as trusted context. — [reverie/backend_server/persona/memory_structures/associative_memory.py:199-201](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/memory_structures/associative_memory.py#L199-L201); [reverie/backend_server/persona/persona.py:44-45](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/persona.py#L44-L45) (verified)
  - *To reach the next level:* No validation, provenance, or gating on memory writes.
- **C L0:** No memory path (events, chats, thoughts, embeddings, forked bootstrap memory) is controlled. — [reverie/backend_server/persona/persona.py:44-45](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/persona.py#L44-L45); [reverie/backend_server/reverie.py:56-58](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/reverie.py#L56-L58) (verified)
  - *To reach the next level:* No memory store is controlled.
- **D L1:** Each persona's memory is a separate folder by path, a code-level separation but not an enforced namespace; there is a single operator. — [reverie/backend_server/persona/persona.py:44-45](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/persona.py#L44-L45) (verified)
  - *To reach the next level:* No enforced namespace or protection against one agent's text being written into another's memory.
- **B L2:** Poisoned memory persists into later forks and influences text output; earlier forks remain as untouched copies that can be resumed. — [reverie/backend_server/reverie.py:56-58](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/reverie.py#L56-L58); [reverie/backend_server/persona/memory_structures/associative_memory.py:199-201](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/memory_structures/associative_memory.py#L199-L201) (verified)
  - *To reach the next level:* No review or rollback step before memory is reused; poisoned text can reach the frontend.
- **Cap:** none

### C7 Third-party extensions — 1.00 (high)

The project loads no third-party code at runtime: no plugins, MCP servers, downloaded tools, model files or package installs. Models are reached only through the OpenAI API. pickle and selenium are imported but never used.

- **Structural absence:** searched `rg -n -S --type py 'pickle\.load|torch\.load|trust_remote_code|importlib|__import__|pip install|entry_points'` in `reverie environment/frontend_server` → 0 hits (No dynamic code loading, model-file deserialization, or package installation.); searched `rg -n -S --type py 'requests\.|urllib|smtplib|webdriver\.|socket\.|http\.client'` in `reverie environment/frontend_server` → 0 hits (No HTTP client, email, socket or browser-automation call besides the OpenAI SDK; selenium is imported but never used.)

### C8 Secrets & sensitive-data protection — 0.17 (high)

The OpenAI key lives in a plaintext Python file the operator creates (git-ignored), and secret handling in the committed Django settings is not locked down. The key is never put into prompts or logs and no telemetry is sent anywhere, but this follows from the design, not from any masking or secret-handling mechanism. The key is a long-lived account key with no scoping.

- **S L0:** Secret handling in the committed default settings is not locked down, and the API key is a plaintext module variable. — [README.md:18](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/README.md#L18) (verified)
  - *To reach the next level:* No masking or secret store.
- **C L1:** By design the key stays out of model messages and there is no logging or telemetry, but no path has an actual redaction mechanism. — [reverie/backend_server/persona/prompt_template/gpt_structure.py:14](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/prompt_template/gpt_structure.py#L14); searched `rg -n -S --type py 'logging\.|logger'` in `reverie environment/frontend_server` → 0 hits (No logging framework; only print() to stdout.) (verified)
  - *To reach the next level:* No redaction on logs, prompts or error output.
- **D L1:** No telemetry; verbose prompt printing is off by default though the README's template sets debug = True, and Django DEBUG is on. — [README.md:32](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/README.md#L32); [environment/frontend_server/frontend_server/settings/base.py:26](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/environment/frontend_server/frontend_server/settings/base.py#L26) (verified)
  - *To reach the next level:* Debug output and Django DEBUG are on in the documented defaults.
- **B L1:** A leak exposes a long-lived OpenAI account key; it is not reachable by the model or any subprocess. — [reverie/backend_server/persona/prompt_template/gpt_structure.py:14](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/prompt_template/gpt_structure.py#L14) (verified)
  - *To reach the next level:* Key is long-lived and not scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.40 (high)

The simulation writes a JSON file for every step with each agent's movement, action description and chat, and saves memory on request. That is a structured, timestamped trail of what agents did, but it omits the prompts and model responses behind them. The files sit in the simulation folder that the local web server can also write to, and the 'exit' command deletes the whole folder. Errors are swallowed, so gaps go unnoticed.

- **S L2:** Per-step movement files record each persona's movement, description, chat and game time in structured JSON. — [reverie/backend_server/reverie.py:400-402](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/reverie.py#L400-L402) (verified)
  - *To reach the next level:* No actor attribution or correlation IDs; prompts and model responses are not recorded.
- **C L2:** Every agent action surfaces in the per-step movement file; memory writes are saved only on save/fin. — [reverie/backend_server/reverie.py:400-402](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/reverie.py#L400-L402) (verified)
  - *To reach the next level:* LLM requests/responses and memory writes are not recorded per action.
- **D L1:** On by default, but stored in the simulation folder that the local web endpoints and the process itself can write. — [reverie/backend_server/reverie.py:400-402](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/reverie.py#L400-L402) (verified)
  - *To reach the next level:* Records are not written outside reach of the process and the local HTTP endpoints.
- **B L1:** Errors are swallowed with bare except blocks and 'exit' deletes the whole simulation folder including its records. — [reverie/backend_server/reverie.py:452-457](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/reverie.py#L452-L457); [reverie/backend_server/persona/prompt_template/gpt_structure.py:222-224](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/prompt_template/gpt_structure.py#L222-L224) (verified)
  - *To reach the next level:* Failures are not surfaced and records can be wiped by a single command.
- **Cap:** none

### C10 Limits & kill switch — 0.30 (high)

Each run is bounded by the step count the operator types (run N), and per-call retries and conversation turns are capped in code. There is no wall-clock limit, no API timeout and no token or cost cap, so a large step count spends without limit. Stopping is by Ctrl-C on a single process, but bare except blocks around the API calls can swallow the interrupt and let the loop continue.

- **S L1:** An iteration cap (operator-chosen step count) plus bounded retries and conversation turns; no time or cost limit. — [reverie/backend_server/reverie.py:464-468](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/reverie.py#L464-L468); [reverie/backend_server/reverie.py:306-309](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/reverie.py#L306-L309); [reverie/backend_server/persona/prompt_template/gpt_structure.py:255-273](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/prompt_template/gpt_structure.py#L255-L273); [reverie/backend_server/persona/cognitive_modules/converse.py:130](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/cognitive_modules/converse.py#L130) (verified)
  - *To reach the next level:* No wall-clock, per-call timeout, or token/cost cap enforced in code.
- **C L1:** The step cap applies to the top-level loop only; model calls have no timeout. — [reverie/backend_server/reverie.py:306-309](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/reverie.py#L306-L309); [reverie/backend_server/persona/prompt_template/gpt_structure.py:255-273](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/prompt_template/gpt_structure.py#L255-L273) (verified)
  - *To reach the next level:* No tool/call timeouts.
- **D L2:** The operator must state the step count each run and the model cannot change it. — [reverie/backend_server/reverie.py:464-468](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/reverie.py#L464-L468) (verified)
  - *To reach the next level:* No hard ceiling on the step count the operator can request.
- **B L1:** Spend is bounded only by the requested step count, and a bare except around API calls swallows KeyboardInterrupt so a stop may not take effect mid-call. — [reverie/backend_server/persona/prompt_template/gpt_structure.py:222-224](https://github.com/joonspk-research/generative_agents/blob/fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4/reverie/backend_server/persona/prompt_template/gpt_structure.py#L222-L224) (verified)
  - *To reach the next level:* No spend ceiling and halt is not reliable while calls are in flight.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Other agents' utterances and forked/loaded memories enter prompts (reverie/backend_server/persona/cognitive_modules/converse.py:146, reverie/backend_server/reverie.py:58) · [B] sensitive data/systems: Low: simulation memories only; the OpenAI key stays in the backend process (reverie/backend_server/persona/prompt_template/gpt_structure.py:14) · [C] state change / egress: Model text shown in the browser frontend and local simulation-state writes · Same default session? Yes

## Highest-impact improvements
1. Harden how the simulator, demo and replay templates render model-written text. — C4 S L0→L3, +0.225 before caps
2. Same change closes the browser exfiltration channel for injected text, limiting a hijacked agent to in-game effects. — C5 B L1→L3, +0.100 before caps (Playbook 1)
3. Harden the local frontend endpoints that read and write simulation state. — C1 C L2→L3, +0.075 before caps (Playbook 4)
4. Add an API request timeout and a per-run token/cost budget, and stop bare except blocks from swallowing KeyboardInterrupt. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
5. Tag memory nodes with their source (self, other agent, operator whisper) and present other-agent content as data when retrieved. — C6 S L0→L2, +0.150 before caps (Playbook 2)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Browser-side consequences of the frontend findings are inferred from standard browser and Django behaviour, not demonstrated.
- Large simulation-data trees (storage/, compressed_storage/, static assets) and defunct_run_gpt_prompt.py were searched but not read line by line.
- The repository is a 2023 research artifact pinned to openai==0.27 and Django 2.2; dependency vulnerabilities are out of scope.
- No text aimed at AI reviewers was found in the repository.
