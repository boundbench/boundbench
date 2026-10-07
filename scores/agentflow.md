# Defense-in-Depth Score: AgentFlow

**Repo:** https://github.com/lupantech/agentflow · **Commit:** `b94006436b8712ab8682846fb0d886a5f174f2d4` · **Reviewed:** 2026-10-04
**What it is:** Trainable modular agent framework (planner, executor, verifier, generator) with web search, Wikipedia and Python tools, plus Flow-GRPO RL training.
**Category:** Agent Frameworks
**Scored configuration:** Inference via construct_solver() with default arguments (enabled_tools=['all'], max_steps=10, max_time=300), as in quick_start.py.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 2.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | Medium |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | Medium |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C9 | Audit & traceability | L1 | L2 | L1 | L0 | 0.28 | — | **0.28** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | Medium |

Controls where a risk surface exists: 1.08 / 9.0 (12%); 1 criterion scored SA (surface absent).

As shipped, AgentFlow runs model-written Python directly inside its own process for every tool call, with no sandbox, no approval step and no argument validation. Web pages and search results are fed into the prompt that writes that code, so a malicious page can steer it to read the provider API keys from the environment and send them anywhere, or damage the host. Only step and time caps exist, and timed-out code keeps running.

## Critical gaps
- Model-generated Python (including code written after reading web content) is exec()'d in-process with no sandbox and no approval, with all API keys in the environment. (ASI05; C4) — [agentflow/agentflow/models/executor.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L226); [agentflow/agentflow/models/executor.py:206](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L206)
- Content from fetched web pages flows into the prompt that writes executed Python, so a hijacking page can leak keys and run arbitrary code unattended. (ASI01, LLM01; C5) — [agentflow/agentflow/tools/web_search/tool.py:149](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/tools/web_search/tool.py#L149); [agentflow/agentflow/models/executor.py:91](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L91); [agentflow/agentflow/models/executor.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L226)

## Criterion details

### C1 Identity & least privilege — 0.05 (medium)

AgentFlow has no agent identity or authorization layer. Importing the engines and tools loads a .env file into the process environment, and every engine and tool builds its own client from those environment keys. Because model-written Python runs inside the same process, any hijacked step can use every provider key and the OS user's full authority.

- **S L0:** Ambient authority: provider API keys are read from os.environ by each engine/tool; no scoping primitive exists. — [agentflow/agentflow/engine/openai.py:24](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/engine/openai.py#L24); [agentflow/agentflow/tools/web_search/tool.py:190](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/tools/web_search/tool.py#L190); searched `rg -n -i 'permission|authoriz|scope'` in `agentflow/agentflow` → 29 hits (all hits are the substring 'dashscope' (provider name); no authorization logic) (verified)
  - *To reach the next level:* No dedicated or scoped identity; L1 needs at least a dedicated identity for agent tools.
- **C L0:** Tools construct their own privileged clients from environment variables; no authorization check on any path. — [agentflow/agentflow/tools/web_search/tool.py:190](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/tools/web_search/tool.py#L190); [agentflow/agentflow/engine/openai.py:94](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/engine/openai.py#L94) (verified)
  - *To reach the next level:* No authorization layer; L1 needs the main tool path checked in code.
- **D L0:** Default install loads all keys from .env into the process at import time with no narrower default. — [agentflow/agentflow/engine/openai.py:24](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/engine/openai.py#L24) (verified)
  - *To reach the next level:* No narrower default identity exists.
- **B L1:** The process holds long-lived keys for several LLM/search providers (spend and data access across multiple services); the framework neither narrows nor widens the OS user's other credentials. — [agentflow/.env.template:15-27](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/.env.template#L15-L27); [agentflow/agentflow/models/executor.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L226) (inferred)
  - *To reach the next level:* Keys are long-lived and span several providers; L2 needs authority limited to one system.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no approval step anywhere. The only check before running a tool is that the model chose a registered tool name; the command itself is model-written Python that runs straight away. Consequential actions, including arbitrary code on the host and outbound requests, happen with no human involved.

- **S L0:** No approval mechanism; the solver only checks the tool name is registered. — [agentflow/agentflow/solver.py:110](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/solver.py#L110); searched `rg -n -i 'approv|confirm|human'` in `agentflow/agentflow` → 3 hits (all hits are Wikipedia demo strings about the human kidney; no approval logic) (verified)
  - *To reach the next level:* No approval at all; L1 needs at least a blanket human approval.
- **C L0:** The most powerful path, exec of model-generated Python, is ungated. — [agentflow/agentflow/models/executor.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L226) (verified)
  - *To reach the next level:* Nothing traverses a gate; L1 needs flagged tools to be gated.
- **D L0:** No approval to enable. — searched `rg -n -i 'approv|confirm|human'` in `agentflow/agentflow` → 3 hits (all hits are Wikipedia demo strings about the human kidney; no approval logic) (verified)
  - *To reach the next level:* Approval does not exist, so it cannot be on by default.
- **B L0:** A wrong step can run arbitrary in-process Python as the OS user (delete files, send data) with no undo or checkpoint. — [agentflow/agentflow/models/executor.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L226); [agentflow/agentflow/models/executor.py:206](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L206) (verified)
  - *To reach the next level:* No rollback or checkpoints; L1 needs some reversible actions.
- **Cap:** none

### C3 Tool & action scoping — 0.00 (high)

Tool 'commands' are not structured arguments: the executor runs whatever Python the model writes, as long as it ends with an execution = tool.execute(...) line, so any code can be prefixed. The web tools fetch any URL with no host allowlist or internal-address block, and the Python tool's only filter strips exit/quit calls. The default constructor enables every bundled tool, including code execution and web fetch.

- **S L0:** Raw passthrough: model-written Python is exec'd, and web tools fetch arbitrary URLs. — [agentflow/agentflow/models/executor.py:206](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L206); [agentflow/agentflow/models/executor.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L226); [agentflow/agentflow/tools/web_search/tool.py:149](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/tools/web_search/tool.py#L149) (verified)
  - *To reach the next level:* No argument validation; L1 needs at least denylist filtering on the exec path.
- **C L0:** Every tool call goes through the same unvalidated exec path; the python_coder exit/quit denylist doesn't constrain arguments. — [agentflow/agentflow/models/executor.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L226); [agentflow/agentflow/tools/python_coder/tool.py:213](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/tools/python_coder/tool.py#L213) (verified)
  - *To reach the next level:* No tool validates its inputs in a way the exec path can't skip.
- **D L0:** construct_solver defaults to enabled_tools=['all'], loading exec and network tools. — [agentflow/agentflow/solver.py:199](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/solver.py#L199); [agentflow/agentflow/models/initializer.py:98](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/initializer.py#L98) (verified)
  - *To reach the next level:* All tools on by default; L1 needs dangerous tools individually disableable by default config.
- **B L0:** General-purpose Python against the whole machine and any host. — [agentflow/agentflow/models/executor.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L226); [agentflow/agentflow/tools/web_search/tool.py:149](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/tools/web_search/tool.py#L149) (verified)
  - *To reach the next level:* No scoping; L1 needs some limit on reach.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

Model-generated code runs inside the agent's own Python process twice over: the executor exec()s every tool command with the executor module's globals, and the Python coder tool exec()s generated code (once with no namespace at all). There is no container, subprocess, or even an import filter, and the process holds every API key and full network access.

- **S L0:** In-process exec() with module globals; no isolation primitive. — [agentflow/agentflow/models/executor.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L226); [agentflow/agentflow/tools/python_coder/tool.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/tools/python_coder/tool.py#L226); searched `rg -n -i 'sandbox|docker|subprocess|seccomp'` in `agentflow/agentflow` → 0 hits (verified)
  - *To reach the next level:* No isolation; L1 needs at least filtering or a restricted executor.
- **C L0:** Neither exec path is sandboxed. — [agentflow/agentflow/models/executor.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L226); [agentflow/agentflow/tools/python_coder/tool.py:237](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/tools/python_coder/tool.py#L237) (verified)
  - *To reach the next level:* No path is sandboxed.
- **D L0:** No sandbox exists to enable. — searched `rg -n -i 'sandbox|docker|subprocess|seccomp'` in `agentflow/agentflow` → 0 hits (verified)
  - *To reach the next level:* No sandbox; it cannot be on by default.
- **B L0:** Escape is moot: code already runs in the agent process with all provider keys in os.environ, the OS user's files, and unrestricted network. — [agentflow/agentflow/models/executor.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L226); [agentflow/agentflow/engine/openai.py:24](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/engine/openai.py#L24) (verified)
  - *To reach the next level:* Host-equivalent; L1 needs at least the credentials kept out of the execution environment.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Web pages, Wikipedia articles, and search results are stored as step results and pasted straight into the planner's and executor's prompts with the same standing as the user's question. The executor then writes Python from that context and runs it. A page that hijacks the model can therefore exfiltrate keys through an arbitrary request and run destructive code, unattended.

- **S L0:** Nothing limits a hijacked agent; no provenance, taint, or capability restriction. — searched `rg -n -i 'untrusted|injection|provenance|taint'` in `agentflow/agentflow` → 1 hits (single hit is a prompt line about analysis limitations); [agentflow/agentflow/models/planner.py:201-202](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/planner.py#L201-L202) (verified)
  - *To reach the next level:* No limit at all; L1 needs at least delimiting or detection of untrusted content.
- **C L0:** Tool results enter the planner and executor prompts undistinguished from user input. — [agentflow/agentflow/models/planner.py:201-202](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/planner.py#L201-L202); [agentflow/agentflow/models/executor.py:91](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L91); [agentflow/agentflow/models/memory.py:65-70](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/memory.py#L65-L70) (verified)
  - *To reach the next level:* Sources aren't distinguished.
- **D L0:** No control to enable. — searched `rg -n -i 'untrusted|injection|provenance|taint'` in `agentflow/agentflow` → 1 hits (single hit is a prompt line about analysis limitations) (verified)
  - *To reach the next level:* No control exists.
- **B L0:** Hijack yields both exfiltration (arbitrary URL fetch, arbitrary Python) and irreversible host actions with no human. — [agentflow/agentflow/tools/web_search/tool.py:149](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/tools/web_search/tool.py#L149); [agentflow/agentflow/models/executor.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L226) (verified)
  - *To reach the next level:* Leak and irreversible action are both unattended; L1 needs one of them blocked.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.30 (medium)

Agent memory lives only for one solve() call, and there are no auto-loaded instruction files. The one persistent store is an opt-in disk cache of LLM responses (off by default in the engine factory) that, when enabled, replays stored responses, including generated tool commands, for any identical prompt in later sessions without validation. The .env file is found relative to the package source, not the working directory (python-dotenv behaviour, inferred).

- **S L1:** When enabled, the response cache stores model outputs keyed by prompt with no validation or provenance and returns them as fresh model output. — [agentflow/agentflow/engine/base.py:27-34](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/engine/base.py#L27-L34); [agentflow/agentflow/engine/openai.py:155-159](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/engine/openai.py#L155-L159) (verified)
  - *To reach the next level:* No provenance or validation on cached entries; L2 needs entries presented as data with provenance.
- **C L1:** The only persistent store is the response cache; Memory is in-process per run. — [agentflow/agentflow/models/memory.py:6-9](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/memory.py#L6-L9); [agentflow/agentflow/engine/factory.py:5](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/engine/factory.py#L5) (verified)
  - *To reach the next level:* No store is controlled; L2 needs the main store controlled.
- **D L2:** Cache is off by default (use_cache=False) and, when on, lives in the OS user's cache directory; the framework is single-user. — [agentflow/agentflow/engine/factory.py:5](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/engine/factory.py#L5); [agentflow/agentflow/engine/openai.py:84](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/engine/openai.py#L84) (verified)
  - *To reach the next level:* No per-session namespacing or guard against the model's in-process code writing the cache; L3 needs the model unable to alter it.
- **B L1:** With the cache on, a poisoned tool-command response replays across the user's sessions and triggers code execution. — [agentflow/agentflow/engine/base.py:27-34](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/engine/base.py#L27-L34); [agentflow/agentflow/models/executor.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L226) (inferred)
  - *To reach the next level:* Poisoned entries persist and can trigger tool use; L2 needs them limited to text or gated actions.
- **Cap:** none
- **Notes:** persistent_memory profiled as opt_in. Model-run code can also write any file (C4), which is scored there, not here.

### C7 Third-party extensions — 1.00 (high)

The inference agent loads no third-party code at runtime: tools are imported only from the package's own tools directory, there is no MCP or plugin loader, and LLM engines are API clients. Training code (verl) defaults trust_remote_code to False. This is absence of the surface, not a control.

- **Structural absence:** searched `rg -n -i 'mcp|plugin|entry_points|trust_remote_code|from_pretrained|hf_hub_download|pip install'` in `agentflow/agentflow` → 11 hits (all 11 hits are ImportError messages telling the user to pip install a provider SDK; no runtime extension loading)
- **Notes:** Scored for the inference solver. agentflow/verl/entrypoint.py:49 reads trust_remote_code from config with default False. Arbitrary in-process exec (C4) could install packages; that is scored under C4.

### C8 Secrets & sensitive-data protection — 0.05 (high)

Secrets are plain environment variables loaded from .env at import, with no masking anywhere. Verbose mode, on by default, prints every prompt, command and tool result to stdout. Model-written code runs in the same process and can read every provider key from os.environ.

- **S L0:** Env-var secrets with no masking or redaction in any path. — [agentflow/agentflow/engine/openai.py:24](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/engine/openai.py#L24); searched `rg -n -i 'redact|mask|secretstr|scrub'` in `agentflow/agentflow` → 0 hits (verified)
  - *To reach the next level:* No masking anywhere; L1 needs masking on at least one path.
- **C L0:** No path (logs, model-bound messages, exec environment) is protected. — searched `rg -n -i 'redact|mask|secretstr|scrub'` in `agentflow/agentflow` → 0 hits; [agentflow/agentflow/solver.py:142](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/solver.py#L142) (verified)
  - *To reach the next level:* No path protected.
- **D L1:** No third-party telemetry in the solver path, but verbose=True by default prints full tool results unredacted. — [agentflow/agentflow/solver.py:207](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/solver.py#L207); [agentflow/agentflow/solver.py:142](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/solver.py#L142) (verified)
  - *To reach the next level:* Verbose payload logging is on by default; L2 needs reasonable logging defaults.
- **B L0:** Long-lived provider keys are reachable by model-generated code via os.environ. — [agentflow/agentflow/models/executor.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L226); [agentflow/agentflow/tools/web_search/tool.py:190](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/tools/web_search/tool.py#L190) (verified)
  - *To reach the next level:* Keys are long-lived and model-reachable; L1 needs them out of model reach.
- **Cap:** none

### C9 Audit & traceability — 0.28 (high)

The solver keeps a per-run dictionary of prompts, generated commands and tool results and returns it to the caller, and prints each step to stdout. Nothing is written to a durable or tamper-resistant log, there is no actor attribution, and the record lives in the same process that runs model code.

- **S L1:** Print statements plus an in-memory json_data dict of commands and results; no timestamps per call or structured audit log. — [agentflow/agentflow/solver.py:142](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/solver.py#L142); [agentflow/agentflow/solver.py:138](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/solver.py#L138) (verified)
  - *To reach the next level:* No durable structured per-call record with timestamps; L2 needs one.
- **C L2:** Every tool call goes through the solver loop, which records command and result. — [agentflow/agentflow/solver.py:136-138](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/solver.py#L136-L138) (verified)
  - *To reach the next level:* No approvals exist to record and no actor attribution; L3 needs approvals/denials and sub-agent paths.
- **D L1:** Record is in the agent's own process memory, which exec'd model code shares. — [agentflow/agentflow/models/executor.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L226) (verified)
  - *To reach the next level:* Record is alterable by model-run code; L2 needs it stored outside the agent's reach.
- **B L0:** The structured record exists only in process memory and is lost if the process crashes; it is persisted only if the caller saves the returned dict. Stdout prints are the only per-step trace and print failures are not surfaced. — [agentflow/agentflow/solver.py:196](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/solver.py#L196); [agentflow/agentflow/solver.py:142](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/solver.py#L142) (verified)
  - *To reach the next level:* Records are lost on crash; L1 needs at least a best-effort persisted record.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (medium)

The solve loop stops after 10 steps or 300 seconds by default, checked between steps, and each tool block waits at most 120 seconds. But timeouts only stop waiting: the daemon thread running model code keeps going, and the Python tool's 10-second timer raises in the wrong thread so it never interrupts anything. There is no token or cost budget, and in-process model code could alter the limits.

- **S L2:** Step cap plus wall-clock cap enforced in the loop, and a per-block wait timeout. — [agentflow/agentflow/solver.py:89](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/solver.py#L89); [agentflow/agentflow/models/executor.py:239-252](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L239-L252) (verified)
  - *To reach the next level:* No token/cost cap or rate limits on side-effecting tools; L3 needs them.
- **C L2:** Top-level loop plus tool-block timeouts. — [agentflow/agentflow/solver.py:89](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/solver.py#L89); [agentflow/agentflow/models/executor.py:239-252](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L239-L252) (verified)
  - *To reach the next level:* Spawned threads don't count against the budget and keep running; L3 needs them covered.
- **D L1:** Defaults are sensible (10 steps, 300 s) but model-written code runs in-process with access to the solver's objects and could change them. — [agentflow/agentflow/solver.py:203-204](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/solver.py#L203-L204); [agentflow/agentflow/models/executor.py:226](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L226) (inferred)
  - *To reach the next level:* The model can reach and alter its limits; L2 needs limits it can't change.
- **B L1:** Timed-out code keeps running in a daemon thread (cooperative cancel event only), and the Python tool's timer does not interrupt exec. — [agentflow/agentflow/models/executor.py:235](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L235); [agentflow/agentflow/models/executor.py:243](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/models/executor.py#L243); [agentflow/agentflow/tools/python_coder/tool.py:30-33](https://github.com/lupantech/agentflow/blob/b94006436b8712ab8682846fb0d886a5f174f2d4/agentflow/agentflow/tools/python_coder/tool.py#L30-L33) (verified)
  - *To reach the next level:* Stopping leaves work running; L2 needs at least the loop ending without orphaned work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web page fetch and Wikipedia/Google search (web_search/tool.py:149, web_rag.py:144) · [B] sensitive data/systems: provider API keys in os.environ loaded via load_dotenv (engine/openai.py:24) · [C] state change / egress: in-process exec of model-written Python (models/executor.py:226) and arbitrary GET (web_search/tool.py:149) · Same default session? Yes

## Highest-impact improvements
1. Stop exec()-ing model text: parse the tool call into a tool name plus JSON arguments and call tool.execute(**args) directly. — C3 S L0→L2, +0.150 before caps (Playbook 3, step 1)
2. Run the Python coder tool in a subprocess or container with no network, no inherited environment, and hard CPU/time limits. — C4 S L0→L3, +0.225 before caps (Playbook 3, step 1)
3. Add an approval hook before code execution and web fetches, on by default for interactive use. — C2 S L0→L3, +0.225 before caps (Playbook 5, step 1)
4. Default enabled_tools to the read-only generator only; require explicit opt-in for Python and web tools. — C3 D L0→L3, +0.150 before caps
5. Block internal addresses and recheck redirects in the web fetch tools. — C3 C L0→L1, +0.075 before caps

## Re-audit log
- No changes.

## Limitations
- Static source review of commit b940064 only; nothing was installed, executed, or probed.
- Framework scored by its defaults (construct_solver); absent primitives score L0 even where a developer could add them.
- Training stack (agentflow/verl, train/, Ray/vLLM, FastAPI task server bound to 127.0.0.1) was surveyed but not scored in depth.
- python-dotenv's find_dotenv search starting at the calling module's directory is library behaviour relied on (inferred) for C6.
- No reviewer-directed prompt injection found in the repo.
