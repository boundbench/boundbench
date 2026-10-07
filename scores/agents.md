# Defense-in-Depth Score: Agents (aiwaves)

**Repo:** https://github.com/aiwaves-cn/agents · **Commit:** `e8c4e3c2d19739d3dff59e577d1c97090cc15f59` · **Reviewed:** 2026-10-04
**What it is:** Python framework for multi-agent language pipelines (SOP graphs of nodes and agents) with symbolic-learning optimizers that rewrite prompts and pipelines.
**Category:** Agent Frameworks
**Scored configuration:** Solution(SolutionConfig(<json>)).run() with default constructor arguments; tools only as listed in the config, with the shipped chatbot example (which enables code_interpreter) used where the framework rules call for official usage.
**Agent surface (default):** code execution opt-in · filesystem write yes · network egress opt-in · external credentials yes · persistent memory yes · untrusted input opt-in · third party extensions no · sub agents yes · external communication opt-in

## Score: 0.4 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L0 | L1 | 0.05 | C6-REPOCONFIG | **0.05** | High |
| C7 | Third-party extensions | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | Medium |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C9 | Audit & traceability | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C10 | Limits & kill switch | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |


This framework executes model tool calls immediately with no approval, validation, sandbox, or run limit. Its flagship code-interpreter tool runs Open Interpreter on the host as your user, with your OpenAI key, so a hijacked or runaway agent has your whole machine. Importing it also loads a .env file and passes two environment values to Python eval, and persisted artifacts do not protect the API key. Treat it as research code, not something to run on untrusted input.

## Critical gaps
- The registered code_interpreter tool runs Open Interpreter on the host as the OS user with the API key, so a hijacked agent holds the user's full local authority. (ASI03, T3; C1) — [src/agents/tools/code_interpreter.py:23-34](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L23-L34); [src/agents/tools/code_interpreter.py:25-27](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L25-L27)
- Model-written code (code_interpreter, sympify, eval of env vars) executes on the host with the user's files, network, and API key; the only guard is documented as not a sandbox. (ASI05, T11; C4) — [src/agents/tools/code_interpreter.py:23-34](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L23-L34); [src/agents/utils/execution.py:88](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/utils/execution.py#L88); [src/agents/agents/memory.py:73](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/memory.py#L73)
- A hijacked agent with the code interpreter can leak secrets and take irreversible host actions with no human in the loop (C5-WORSTCASE). (ASI01, LLM01, T6; C5) — [src/agents/tools/code_interpreter.py:23-34](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L23-L34); [src/agents/agents/agent.py:134-141](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent.py#L134-L141)
- load_dotenv() at import plus eval() of environment values lets a planted .env redirect the model endpoint or run arbitrary Python, with no trust decision (C6-REPOCONFIG). (ASI06, T1; C6) — [src/agents/agents/llm.py:27](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/llm.py#L27); [src/agents/agents/memory.py:73](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/memory.py#L73); [src/agents/agents/agent_team.py:145-147](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent_team.py#L145-L147)
- Packages installed at the model's request through the code interpreter run as the user with the full environment and API key. (ASI04, T17; C7) — [src/agents/tools/code_interpreter.py:23-34](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L23-L34); [src/agents/tools/code_interpreter.py:25-27](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L25-L27)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

The framework has no agent identity or authorization layer. Every agent and tool runs with the developer's OS user and environment; the code-interpreter tool hands Open Interpreter the OPENAI_API_KEY and runs it in-process with full access to the host. Nothing narrows what a hijacked agent can reach.

- **S L0:** Ambient authority only: the OpenAI key is read from the process environment and the code interpreter runs as the OS user. — [src/agents/agents/llm.py:60-61](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/llm.py#L60-L61); [src/agents/tools/code_interpreter.py:25-27](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L25-L27); searched `rg -n -S 'authoriz|permission|scope' -t py` in `src` → 25 hits (hits are Apache license headers and Gmail OAuth SCOPES in the unregistered mail tool; no authorization layer) (verified)
  - *To reach the next level:* No dedicated or scoped identity for agents or tools.
- **C L0:** The tool executor calls tool.func directly with no authorization check. — [src/agents/agents/agent.py:253](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent.py#L253) (verified)
  - *To reach the next level:* No authorization check on the main tool path.
- **D L0:** Defaults use the operator's full environment; no least-privilege option exists. — [src/agents/agents/llm.py:60-61](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/llm.py#L60-L61) (verified)
  - *To reach the next level:* No narrower default identity exists to enable.
- **B L0:** With the code interpreter enabled (as in the chatbot example) a hijacked agent acts with the user's whole account on the host. — [src/agents/tools/code_interpreter.py:23-34](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L23-L34); [examples/chatbot/config.json:26-29](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/examples/chatbot/config.json#L26-L29) (verified)
  - *To reach the next level:* Nothing confines tools below the OS user's authority.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no approval step anywhere. The agent loop parses the model's tool call and executes it immediately, including the code interpreter that writes and runs arbitrary code. The framework offers no hook for a human to review or reject a call, so every consequential action is unattended.

- **S L0:** No approval mechanism: tool calls are executed as soon as the model emits them. — [src/agents/agents/agent.py:234](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent.py#L234); [src/agents/agents/agent.py:253](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent.py#L253); searched `rg -n -S 'approv|confirm|human_in_the_loop|allowlist|whitelist' -t py` in `src` → 2 hits (both hits are comments about confirming vector-DB collection names; no tool gating) (verified)
  - *To reach the next level:* No per-call human approval exists.
- **C L0:** The most powerful tool (code_interpreter) is ungated, as is every other tool. — [src/agents/agents/agent.py:253](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent.py#L253); [src/agents/tools/__init__.py:32-37](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/__init__.py#L32-L37) (verified)
  - *To reach the next level:* No gate exists for any tool path.
- **D L0:** There is no approval to turn on. — searched `rg -n -S 'approv|confirm|human_in_the_loop|allowlist|whitelist' -t py` in `src` → 2 hits (both hits are comments about confirming vector-DB collection names; no tool gating) (verified)
  - *To reach the next level:* Approval is not available even as an option.
- **B L0:** Executed code can delete files or send data irreversibly with no checkpoint or undo. — [src/agents/tools/code_interpreter.py:23-34](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L23-L34) (verified)
  - *To reach the next level:* No rollback, preview, or rate limit on consequential actions.
- **Cap:** none

### C3 Tool & action scoping — 0.05 (high)

Tools receive the model's arguments with no validation in code: the JSON arguments are splatted straight into the tool function. The built-in tools are general-purpose (free-form code requests, sympy expressions that sympify evaluates). The one real narrowing is that toolkits are built only from named entries in a fixed registry and the agent rejects tool names it was not given.

- **S L0:** Raw passthrough: arguments go straight from json.loads to tool.func, and the main tool accepts a free-form coding request. — [src/agents/agents/agent.py:234](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent.py#L234); [src/agents/agents/agent.py:253](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent.py#L253); [src/agents/tools/code_interpreter.py:23-34](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L23-L34) (verified)
  - *To reach the next level:* No typed validation, allowlists, or bounds on tool arguments.
- **C L0:** No tool validates its inputs. — [src/agents/agents/agent.py:253](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent.py#L253); [src/agents/tools/math.py:27](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/math.py#L27) (verified)
  - *To reach the next level:* No tool or shared layer validates arguments.
- **D L1:** Agents get no tools unless the config lists them, and only names in AVAILABLE_TOOLS can be built; but the registry's flagship tool is unrestricted code execution and the chatbot example enables it. — [src/agents/agents/toolkit.py:44-60](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/toolkit.py#L44-L60); [src/agents/tools/__init__.py:32-37](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/__init__.py#L32-L37); [examples/chatbot/config.json:26-29](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/examples/chatbot/config.json#L26-L29) (verified)
  - *To reach the next level:* No read-only default set or per-task tool allowlists beyond the config; capped one level above S.
- **B L0:** The code interpreter is a general-purpose tool against the whole machine. — [src/agents/tools/code_interpreter.py:23-34](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L23-L34) (verified)
  - *To reach the next level:* Tools are not scoped to a workspace or bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

Model-influenced code runs without isolation on several paths: the code-interpreter tool runs Open Interpreter on the host, the math tools pass model strings to sympy's eval-based parser, and two settings read from environment variables are passed to Python eval. The HumanEval scorer runs generated code in a subprocess with a function-disabling guard whose own docstring says it is not a security sandbox. Nothing contains an escape, and the host process holds the API key.

- **S L0:** The main execution path (Open Interpreter) runs on the host as the same user; the only other mechanism is an in-process function-nulling guard explicitly documented as not a sandbox. — [src/agents/tools/code_interpreter.py:23-34](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L23-L34); [src/agents/utils/execution.py:88](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/utils/execution.py#L88); searched `rg -n -S 'docker|sandbox|seccomp|container' -t py` in `src` → 5 hits (hits are Azure storage-container docstrings and the reliability_guard disclaimer that it is not a sandbox) (verified)
  - *To reach the next level:* No OS-level separation (container, low-privilege user) for any execution path.
- **C L0:** Only the HumanEval scorer applies the guard; the code interpreter, sympify, and env-var eval paths have nothing. — [src/agents/datasets/humaneval.py:99](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/datasets/humaneval.py#L99); [src/agents/tools/math.py:27](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/math.py#L27); [src/agents/agents/memory.py:73](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/memory.py#L73); [src/agents/agents/agent_team.py:145-147](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent_team.py#L145-L147) (verified)
  - *To reach the next level:* The main exec tool is not behind any boundary.
- **D L0:** No sandbox exists to be on by default. — searched `rg -n -S 'docker|sandbox|seccomp|container' -t py` in `src` → 5 hits (hits are Azure storage-container docstrings and the reliability_guard disclaimer that it is not a sandbox) (verified)
  - *To reach the next level:* No isolation available even as an option.
- **B L0:** Host-equivalent: code runs as the user with the API key in the environment and full network. — [src/agents/tools/code_interpreter.py:23-34](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L23-L34); [src/agents/tools/code_interpreter.py:25-27](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L25-L27) (verified)
  - *To reach the next level:* Execution is not confined away from host files, credentials, and network.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Nothing separates untrusted content from instructions. Peer agents' messages are folded into a user-role observation, tool results become the agent's next content, and retrieved knowledge-base or memory text goes straight into prompts. With the code interpreter enabled, a hijacked agent can both exfiltrate data and take irreversible actions with no human involved.

- **S L0:** No structural limit on a hijacked agent; there is not even a delimiter or classifier. — [src/agents/agents/agent.py:134-141](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent.py#L134-L141); [src/agents/agents/agent.py:253](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent.py#L253); [src/agents/tools/code_interpreter.py:23-34](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L23-L34) (verified)
  - *To reach the next level:* No approval or capability restriction after untrusted content is read.
- **C L0:** Tool results, peer messages, and retrieved memory enter context with the same standing as instructions. — [src/agents/agents/agent.py:134-141](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent.py#L134-L141); [src/agents/task/sop.py:418-420](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/task/sop.py#L418-L420) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished at all.
- **D L0:** No control exists to be on by default. — [src/agents/agents/agent.py:134-141](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent.py#L134-L141) (verified)
  - *To reach the next level:* No control to enable.
- **B L0:** With the code interpreter (chatbot example) a hijacked agent can read secrets, send them anywhere, and modify the host unattended. — [src/agents/tools/code_interpreter.py:23-34](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L23-L34); [examples/chatbot/config.json:26-29](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/examples/chatbot/config.json#L26-L29); [src/agents/tools/code_interpreter.py:25-27](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L25-L27) (verified)
  - *To reach the next level:* Some leg of the Rule of Two must be removed or gated in code.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.05 (high)

Agent and shared environment long-term memory are appended to JSONL files under a relative memory/ directory that is never cleared, so content from one run persists into the next and is retrieved into routing prompts when a knowledge base is configured. The LLM module calls load_dotenv() at import, so a .env file found by python-dotenv's search can redirect the model endpoint and key, and two settings from the environment are passed to Python eval, turning a planted .env into code execution. There is no validation, provenance, or isolation on any of it.

- **S L0:** Model output is appended to persistent memory unvalidated, and an auto-loaded .env can redirect OPENAI_BASE_URL and inject code via eval of TOP_K / ENVIRONMENT_SUMMARY_STEP. — [src/agents/utils/storages/key_value_storages/json.py:29](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/utils/storages/key_value_storages/json.py#L29); [src/agents/agents/llm.py:27](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/llm.py#L27); [src/agents/agents/llm.py:35](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/llm.py#L35); [src/agents/agents/memory.py:73](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/memory.py#L73) (verified)
  - *To reach the next level:* No validation or provenance on memory writes; environment files load silently.
- **C L0:** No memory store or auto-loaded file is controlled. — [src/agents/agents/memory.py:189-191](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/memory.py#L189-L191); [src/agents/agents/llm.py:27](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/llm.py#L27) (verified)
  - *To reach the next level:* No store or config path is controlled.
- **D L0:** Memory paths are fixed relative files (memory/<agent>.jsonl, memory/environment.jsonl) shared by every run in the same directory. — [src/agents/agents/environment.py:58](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/environment.py#L58); [src/agents/agents/memory.py:189-191](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/memory.py#L189-L191) (verified)
  - *To reach the next level:* No per-user or per-session namespace.
- **B L1:** Poisoned memory persists across runs and is retrieved into the transit prompt that picks the next node and acting agent. — [src/agents/task/sop.py:418-420](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/task/sop.py#L418-L420); [src/agents/utils/storages/key_value_storages/json.py:29](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/utils/storages/key_value_storages/json.py#L29) (verified)
  - *To reach the next level:* Persistent memory is not confined to text output or gated actions.
- **Cap:** C6-REPOCONFIG — load_dotenv() runs at import (llm.py:27) and the code then reads OPENAI_BASE_URL and evals TOP_K/ENVIRONMENT_SUMMARY_STEP from the environment; python-dotenv's default find_dotenv walks up from the calling file's directory, so a .env in the project tree (e.g. a project-local venv or editable install) silently redirects the model endpoint or executes code (library behaviour inferred).

### C7 Third-party extensions — 0.05 (medium)

The framework itself loads no plugins, MCP servers, or model files. The exposure is through the code interpreter: Open Interpreter can install whatever packages the model asks for, unpinned and unverified, and they run as the user with the full environment. That tool is off unless configured, but it is the registry's flagship tool and the chatbot example enables it.

- **S L0:** Model-chosen package installs are possible through the code interpreter with no pinning or verification (Open Interpreter executes arbitrary install commands; inferred from its documented behaviour). — [src/agents/tools/code_interpreter.py:23-34](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L23-L34); searched `rg -n -S 'importlib|entry_points|trust_remote_code|torch\.load|pickle|mcp|from_pretrained|__import__' -t py` in `src` → 0 hits (no plugin loader, MCP client, or model-file deserialisation) (inferred)
  - *To reach the next level:* No pinning or verification of anything installed at the model's request.
- **C L0:** No extension type is verified. — searched `rg -n -S 'importlib|entry_points|trust_remote_code|torch\.load|pickle|mcp|from_pretrained|__import__' -t py` in `src` → 0 hits (no plugin loader, MCP client, or model-file deserialisation) (verified)
  - *To reach the next level:* No verification on any path.
- **D L1:** Nothing third-party is enabled until a config adds the code interpreter, but adding it grants unrestricted installs without showing what will run. — [src/agents/agents/toolkit.py:44-60](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/toolkit.py#L44-L60); [examples/chatbot/config.json:26-29](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/examples/chatbot/config.json#L26-L29) (verified)
  - *To reach the next level:* Installs are not shown or approved; capped one level above S.
- **B L0:** Installed packages run as the same user with the full environment, including the API key. — [src/agents/tools/code_interpreter.py:25-27](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L25-L27) (verified)
  - *To reach the next level:* No separate process or scrubbed environment for installed code.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.05 (high)

The OpenAI key is read from the environment, but persisted artifacts do not protect the key. Logs store full prompt messages in plaintext with no redaction, the flagship example turns on litellm verbose logging, and the Trainer starts Weights & Biases by default. The key is long-lived and visible to every code-interpreter process.

- **S L0:** No masking anywhere, and persisted artifacts do not protect the API key. — searched `rg -n -S 'redact|mask|SecretStr' -t py` in `src` → 0 hits (no redaction or masking anywhere) (verified)
  - *To reach the next level:* No masking or redaction on any path.
- **C L0:** No path is protected: logs, trajectories, and subprocess environments all carry raw content. — [src/agents/utils/files.py:41](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/utils/files.py#L41); searched `rg -n -S 'redact|mask|SecretStr' -t py` in `src` → 0 hits (no redaction or masking anywhere) (verified)
  - *To reach the next level:* No redaction on any path.
- **D L1:** Trainer calls wandb.init unconditionally (content-free metrics and trainer config); the chatbot example sets litellm.set_verbose = True. — [src/agents/optimization/trainer.py:127](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/optimization/trainer.py#L127); [examples/chatbot/run.py:7](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/examples/chatbot/run.py#L7) (verified)
  - *To reach the next level:* Telemetry is not opt-in; capped one level above S.
- **B L0:** A long-lived, unscoped OpenAI key is reachable by the model's code and every subprocess. — [src/agents/tools/code_interpreter.py:25-27](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/tools/code_interpreter.py#L25-L27); [src/agents/agents/llm.py:60-61](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/llm.py#L60-L61) (verified)
  - *To reach the next level:* Credentials are long-lived and unscoped.
- **Cap:** none

### C9 Audit & traceability — 0.20 (high)

The only record is a JSON dump of the prompt messages and the final content, written to a relative logs/ directory when a tool is called (and on other calls only if SAVE_LOGS is set). It does not record which tool ran or with what arguments, and the logger deletes all but the newest 20 files in the directory. Training mode keeps fuller trajectories, but normal runs leave no reliable audit trail.

- **S L1:** Unstructured per-call dumps of messages and output; tool names and arguments are not recorded. — [src/agents/agents/agent.py:256](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent.py#L256); [src/agents/utils/files.py:41](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/utils/files.py#L41) (verified)
  - *To reach the next level:* No structured record of each tool call with arguments and status.
- **C L1:** Only the agent tool-call path writes a log unconditionally; other LLM calls log only when SAVE_LOGS is set. — [src/agents/agents/agent.py:256](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent.py#L256); [src/agents/agents/llm.py:68](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/llm.py#L68) (verified)
  - *To reach the next level:* Not all calls are recorded by default.
- **D L1:** Written by default to a relative logs/ path inside the working directory, which the code interpreter can edit. — [src/agents/agents/agent.py:256](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/agent.py#L256); [src/agents/utils/files.py:38](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/utils/files.py#L38) (verified)
  - *To reach the next level:* Logs are not stored outside the agent's reach.
- **B L0:** Records are silently deleted once more than 20 files exist, and failures are not surfaced. — [src/agents/utils/files.py:38](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/utils/files.py#L38) (verified)
  - *To reach the next level:* Records are not durable per action.
- **Cap:** none

### C10 Limits & kill switch — 0.00 (high)

A run has no limit on steps, time, or cost. Solution.run loops until the SOP reaches its end node; the per-node max_chat_nums check is overwritten by the LLM or order transition on the next line, single-successor nodes loop forever, and the LLM call retries forever on errors. Only the training loop (max_step) and the HumanEval scorer (3-second timeout with process kill) are bounded.

- **S L0:** No enforced step, time, or cost limit on a run; the only cap (max_chat_nums) is dead code and retries are infinite. — [src/agents/task/solution.py:81](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/task/solution.py#L81); [src/agents/task/sop.py:220-227](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/task/sop.py#L220-L227); [src/agents/task/sop.py:215-216](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/task/sop.py#L215-L216); [src/agents/agents/llm.py:40-45](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/llm.py#L40-L45); searched `rg -n -S 'max_steps|max_turns|max_iterations|recursion_limit|timeout' -t py` in `src` → 8 hits (all hits are in the HumanEval evaluator and its time_limit helper; nothing bounds the agent run loop) (verified)
  - *To reach the next level:* No iteration cap that actually stops a run.
- **C L0:** Limits apply to nothing in the agent loop. — [src/agents/task/solution.py:81](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/task/solution.py#L81) (verified)
  - *To reach the next level:* No limit covers the top-level loop.
- **D L0:** Unlimited by default. — [src/agents/task/solution.py:81](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/task/solution.py#L81) (verified)
  - *To reach the next level:* No default limit exists.
- **B L0:** A runaway can loop and spend indefinitely; nothing stops in-flight interpreter work. — [src/agents/agents/llm.py:40-45](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/agents/llm.py#L40-L45); [src/agents/task/sop.py:215-216](https://github.com/aiwaves-cn/agents/blob/e8c4e3c2d19739d3dff59e577d1c97090cc15f59/src/agents/task/sop.py#L215-L216) (verified)
  - *To reach the next level:* No ceiling on spend or time.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: peer/tool/retrieved content enters prompts unmarked (src/agents/agents/agent.py:134-141, src/agents/task/sop.py:418) · [B] sensitive data/systems: OPENAI_API_KEY in process env and passed to the interpreter (src/agents/tools/code_interpreter.py:25-27) · [C] state change / egress: code_interpreter runs arbitrary code with network on the host (src/agents/tools/code_interpreter.py:34) · Same default session? Yes

## Highest-impact improvements
1. Enforce a hard step/wall-clock cap in Solution.run, fix the overwritten max_chat_nums branch, and bound LLM retries. — C10 S L0→L2, +0.150 before caps (Playbook 3 step 3)
2. Add an approval callback in Agent.step before tool.func showing the exact tool name and arguments, on by default for code_interpreter. — C2 S L0→L3, +0.225 before caps (Playbook 5)
3. Drop the import-time load_dotenv() and replace eval() of environment values with int(). — C6 S L0→L1, +0.075 before caps (Playbook 2)
4. Keep credentials out of persisted artifacts and redact secrets from saved logs and trajectories. — C8 S L0→L1, +0.075 before caps (Playbook 4)
5. Run the code interpreter in a hardened container with a scrubbed environment by default. — C4 S L0→L3, +0.225 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Open Interpreter 0.2.5 behaviour (no confirmation when called programmatically via chat(display=False), ability to pip install) is inferred from that library, not read in this repo.
- python-dotenv find_dotenv search behaviour (walks up from the calling file's directory) is inferred from the library.
- Unregistered tools (web_search, web_crawl, mail, translate, weather) are importable but not reachable via Toolkit config; reviewed but not scored as defaults. Datasets under src/agents/datasets/data were not read.
- No text aimed at AI reviewers was found (searched for auditor/AI-reviewer phrasing).
