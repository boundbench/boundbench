# Defense-in-Depth Score: AgentVerse

**Repo:** https://github.com/openbmb/agentverse · **Commit:** `f90c4bd9680fdd3bcff8c52c9170911a59b23478` · **Reviewed:** 2026-10-04
**What it is:** OpenBMB's Python framework for multi-LLM-agent task solving (role assignment, decision making, execution, evaluation) and multi-agent simulations.
**Category:** Agent Frameworks
**Scored configuration:** Framework defaults as shipped: the agentverse-tasksolving / agentverse-benchmark CLIs with the bundled task configs (default task tasksolving/brainstorming; humaneval code-test and tool-using executors as shipped), OPENAI_API_KEY in the environment.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions opt-in · sub agents yes · external communication no

## Score: 2.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L2 | 0.28 | — | **0.28** | Medium |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C9 | Audit & traceability | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C10 | Limits & kill switch | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |

Controls where a risk surface exists: 1.05 / 9.0 (12%); 1 criterion scored SA (surface absent).

AgentVerse ships no safety controls around what its agents do. Its built-in executors run model-generated code on the host through a shell or Python eval/exec, with the operator's API keys in the environment and no approval, sandbox, or argument checks. Tool-using tasks forward any tool call the model emits, including a root shell, to an external tool server. Treat it as a research framework and run it only inside a disposable VM or container that holds no credentials you care about.

## Critical gaps
- Model-generated code runs on the host as the invoking user through shell=True, with the full environment (including OPENAI_API_KEY) inherited. (ASI05, T11; C4) — [agentverse/environments/tasksolving_env/rules/executor/code_test.py:18](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L18); [agentverse/llms/openai.py:38](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/llms/openai.py#L38)
- A hijacked agent holds the operator's ambient identity: model-run subprocesses inherit all credentials in the launching shell. (ASI03, T3; C1) — [agentverse/environments/tasksolving_env/rules/executor/code_test.py:18](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L18); [agentverse/llms/openai.py:38](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/llms/openai.py#L38)
- Untrusted web content, an ungated root shell tool and host code execution coexist with no approval; injected content can exfiltrate and act irreversibly unattended (C5-WORSTCASE). (ASI01, LLM01, T6; C5) — [agentverse/environments/tasksolving_env/rules/executor/tool_using.py:329-333](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/tool_using.py#L329-L333); [agentverse/tasks/tasksolving/tool_using/tools_simplified.json:204-205](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/tasks/tasksolving/tool_using/tools_simplified.json#L204-L205); [agentverse/environments/tasksolving_env/rules/executor/tool_using.py:318-327](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/tool_using.py#L318-L327)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

AgentVerse reads the operator's OpenAI or Azure API key from the environment and has no notion of a scoped agent identity or per-request authorization. Model-written code run by the code-test executor is launched with the default subprocess environment, so it inherits that key and every other credential in the operator's shell. If an agent is hijacked, it acts with the full authority of the user who launched it.

- **S L0:** The only identity is the operator's ambient environment: long-lived API keys read from os.environ and whatever else the OS user holds. — [agentverse/llms/openai.py:38](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/llms/openai.py#L38); [agentverse/llms/openai.py:40](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/llms/openai.py#L40) (verified)
  - *To reach the next level:* No dedicated or scoped identity for the agent or its tools.
- **C L0:** There is no authorization layer at all; tools and executors run with whatever the process holds, and subprocesses inherit the full environment. — [agentverse/environments/tasksolving_env/rules/executor/code_test.py:18](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L18); searched `rg -n "env="` in `agentverse/environments agentverse/agents` → 1 hits (only hit is httpx trust_env=True in tool_using.py:303; no subprocess call passes a scrubbed env) (verified)
  - *To reach the next level:* No authorization check on any tool path; subprocesses are not given a scrubbed environment.
- **D L0:** The default install runs every executor as the invoking OS user with the full environment. — [agentverse/environments/tasksolving_env/rules/executor/code_test.py:18](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L18); [agentverse/llms/openai.py:38](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/llms/openai.py#L38) (verified)
  - *To reach the next level:* No narrower default role; least privilege would require the operator to build it externally.
- **B L0:** A hijacked agent running model-written shell code reaches the operator's whole account: API keys, SSH keys, cloud credentials, and all files the user can touch. — [agentverse/environments/tasksolving_env/rules/executor/code_test.py:18](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L18); searched `rg -n "env="` in `agentverse/environments agentverse/agents` → 1 hits (only hit is httpx trust_env=True in tool_using.py:303; no subprocess call passes a scrubbed env) (verified)
  - *To reach the next level:* Nothing limits the reach of a hijacked executor below the full OS user.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no human approval step anywhere in the framework. Model-written code is executed, files are written to model-chosen paths, and tool calls (including a root shell tool exposed by the tool-using configs) are sent to the tool server without asking anyone. The only input() prompts in the code are commented out and were for scoring, not gating.

- **S L0:** No approval mechanism exists; actions proceed directly from model output. — searched `rg -n -S "input\(|confirm|approv"` in `agentverse agentverse_command` → 5 hits (all 5 hits are commented-out input() prompts for human evaluation scores in rules/base.py:151-155; none gate an action) (verified)
  - *To reach the next level:* No per-call human approval of any kind.
- **C L0:** The most powerful paths, host shell execution and arbitrary tool-server calls, are ungated. — [agentverse/environments/tasksolving_env/rules/executor/code_test.py:18](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L18); [agentverse/environments/tasksolving_env/rules/executor/tool_using.py:318-327](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/tool_using.py#L318-L327) (verified)
  - *To reach the next level:* Shell and tool execution paths would need to traverse a gate.
- **D L0:** There is nothing to turn on; approval is absent by default. — searched `rg -n -S "input\(|confirm|approv"` in `agentverse agentverse_command` → 5 hits (all 5 hits are commented-out input() prompts for human evaluation scores in rules/base.py:151-155; none gate an action) (verified)
  - *To reach the next level:* No default-on approval.
- **B L0:** Wrongly taken actions include arbitrary host shell commands and file overwrites with no checkpoint or undo. — [agentverse/environments/tasksolving_env/rules/executor/code_test.py:18](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L18); [agentverse/environments/tasksolving_env/rules/executor/code_test.py:43](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L43) (verified)
  - *To reach the next level:* No rollback, checkpoints, or quantity limits on consequential actions.
- **Cap:** none

### C3 Tool & action scoping — 0.05 (high)

Tools are as broad as they can be: the code-test executor writes model-generated code to a model-chosen file path with no containment and then runs it through a shell, and the tool-using executor forwards whatever tool name and arguments the model emits to the tool server without checking them against the configured tool list. Shipped tool configs include a root shell, a Python notebook, file writes and web browsing. Only the simulation ToolAgent checks that a named tool exists, which is not argument validation.

- **S L0:** Raw passthrough: arbitrary shell strings, arbitrary file paths, and arbitrary tool names/arguments. — [agentverse/environments/tasksolving_env/rules/executor/code_test.py:43](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L43); [agentverse/environments/tasksolving_env/rules/executor/tool_using.py:318-327](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/tool_using.py#L318-L327); [agentverse/environments/tasksolving_env/rules/executor/tool_using.py:279-290](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/tool_using.py#L279-L290) (verified)
  - *To reach the next level:* No allowlist or containment validation on any argument.
- **C L0:** No tool validates its inputs; the tool-using executor does not even check the tool name against the configured list. — [agentverse/environments/tasksolving_env/rules/executor/tool_using.py:318-327](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/tool_using.py#L318-L327); searched `rg -n "tool_names"` in `agentverse/environments/tasksolving_env/rules/executor/tool_using.py` → 6 hits (tool_names is built at init (lines 34,47,64) and used only for retrieval de-duplication (206-210); call_tool never consults it) (verified)
  - *To reach the next level:* At least some tools would need argument validation.
- **D L1:** Tool sets are chosen per task YAML, so dangerous tools can be removed, but the shipped tool-using config enables a root shell, notebook execution and file writes. — [agentverse/tasks/tasksolving/tool_using/tools_simplified.json:204-205](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/tasks/tasksolving/tool_using/tools_simplified.json#L204-L205); [agentverse/tasks/tasksolving/tool_using/24point/config.yaml:5](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/tasks/tasksolving/tool_using/24point/config.yaml#L5) (verified)
  - *To reach the next level:* Default tool groups still include write and exec; no read-only default set.
- **B L0:** A misused tool reaches the whole machine: any shell command as the host user, or any command as root inside the tool server. — [agentverse/environments/tasksolving_env/rules/executor/code_test.py:18](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L18); [agentverse/tasks/tasksolving/tool_using/tools_simplified.json:204-205](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/tasks/tasksolving/tool_using/tools_simplified.json#L204-L205) (verified)
  - *To reach the next level:* No workspace scoping or quantity bounds on tools.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

Model-generated code runs with no isolation. The code-test executor runs it through a host shell as the current user; the simulation software-team environment exec()s model-written code and evals model-written test lists in-process; one further in-process path is not confined either. The only isolation anywhere is the external XAgent ToolServer container used by the tool-using tasks, which is not part of this repository and could not be verified. An escape is not needed: the code already has the user's files, network and API keys.

- **S L0:** Execution is same-user host subprocess, in-process exec(), and eval() of model output. — [agentverse/environments/tasksolving_env/rules/executor/code_test.py:18](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L18); [agentverse/environments/simulation_env/rules/selector/code_api.py:38](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/simulation_env/rules/selector/code_api.py#L38) (verified)
  - *To reach the next level:* No OS-level or runtime isolation primitive.
- **C L0:** No execution path is sandboxed. — [agentverse/environments/tasksolving_env/rules/executor/code_test.py:18](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L18); [agentverse/environments/simulation_env/rules/selector/sde_team.py:55](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/simulation_env/rules/selector/sde_team.py#L55); searched `rg -n -i "docker|sandbox|seccomp|chroot"` in `agentverse` → 2 hits (both hits are comments in tool_using.py about the external ToolServer container being slow to start; no sandbox code exists in this repo) (verified)
  - *To reach the next level:* At least the main exec path would need a sandbox.
- **D L0:** There is no sandbox to enable. — [agentverse/environments/tasksolving_env/rules/executor/code_test.py:18](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L18) (verified)
  - *To reach the next level:* No sandbox on by default.
- **B L0:** Executed code has the full host user's filesystem, unrestricted network and the inherited API keys in its environment. — [agentverse/environments/tasksolving_env/rules/executor/code_test.py:18](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L18); [agentverse/llms/openai.py:38](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/llms/openai.py#L38); searched `rg -n "env="` in `agentverse/environments agentverse/agents` → 1 hits (only hit is httpx trust_env=True in tool_using.py:303; no subprocess call passes a scrubbed env) (verified)
  - *To reach the next level:* Code would need a workspace-only, no-secret, egress-limited environment.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Nothing distinguishes untrusted content from instructions. Web pages fetched through the tool server are summarized and fed back into agent memory as ordinary function messages, and messages from other agents in the group are treated as context with the same standing. Because the framework also executes model output and has network egress with no approval, a successful injection in a tool-using or code-execution task can both exfiltrate data and take irreversible actions unattended.

- **S L0:** No structural limit on a hijacked agent; there are not even delimiters or detection. — searched `rg -n -i "untrusted|injection"` in `agentverse agentverse_command` → 0 hits; [agentverse/environments/tasksolving_env/rules/executor/tool_using.py:329-333](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/tool_using.py#L329-L333) (verified)
  - *To reach the next level:* No approval or capability restriction after untrusted content is read.
- **C L0:** Tool results enter context as normal function-role messages, and peer-agent messages are not separated from the principal's task. — [agentverse/memory/chat_history.py:93-99](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/memory/chat_history.py#L93-L99); [agentverse/environments/tasksolving_env/rules/executor/tool_using.py:161](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/tool_using.py#L161) (verified)
  - *To reach the next level:* Untrusted sources would need to be distinguished.
- **D L0:** No control exists to be on by default. — searched `rg -n -i "untrusted|injection"` in `agentverse agentverse_command` → 0 hits (verified)
  - *To reach the next level:* No default-on containment for untrusted content.
- **B L0:** Per the framework rule, shipped tool-using tasks combine web content, a root shell and file tools, and code-execution tasks run model output on the host with API keys in the environment; a hijack can leak secrets and take irreversible actions with no human involved. — [agentverse/environments/tasksolving_env/rules/executor/tool_using.py:323-327](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/tool_using.py#L323-L327); [agentverse/tasks/tasksolving/tool_using/tools_simplified.json:204-205](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/tasks/tasksolving/tool_using/tools_simplified.json#L204-L205); [agentverse/environments/tasksolving_env/rules/executor/code_test.py:18](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L18) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions would need human approval.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 1.00 (high)

No persistence path exists that the model can influence. Agent memory (chat history, summaries, the in-memory vector store, reflections) lives only in the running process and is rebuilt for every task and every benchmark example. Task configuration comes from the package's own tasks directory or an explicit --tasks_dir flag, and nothing loads .env files or instruction files from the working directory. Caveat: because model-written code runs unsandboxed on the host (C4), it could still overwrite files such as task configs; that is scored as execution blast radius, not as a memory feature.

- **Structural absence:** searched `rg -n "dotenv|AGENTS\.md|CLAUDE\.md|pickle|faiss|chroma|sqlite"` in `agentverse agentverse_command` → 0 hits; searched `rg -n "json\.dump|open\("` in `agentverse/memory agentverse/memory_manipulator` → 1 hits (single hit is json.dumps of tool_input when building model messages (chat_history.py:81); no memory is written to disk)

### C7 Third-party extensions — 0.28 (medium)

Third-party tools are not loaded by default. Simulation tasks can list BMTools tool servers by URL in the task YAML, and tool-using tasks call an external XAgent ToolServer over HTTP; in both cases the spec and tool list are fetched live with no pinning or integrity check. These servers run as separate processes the operator starts themselves, and AgentVerse sends them only tool arguments and session cookies, not its API keys.

- **S L1:** Tool sources are operator-chosen URLs, refetched on every launch with no version pin or integrity check. — [agentverse/initialization.py:49-56](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/initialization.py#L49-L56); [agentverse/tasks/simulation/math_problem_2players_tools/config.yaml:70-71](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/tasks/simulation/math_problem_2players_tools/config.yaml#L70-L71) (verified)
  - *To reach the next level:* No version pinning of tool specs or tool lists.
- **C L0:** No extension type (BMTools servers, ToolServer tools, retrieved tools) is verified. — [agentverse/environments/tasksolving_env/rules/executor/tool_using.py:200-211](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/tool_using.py#L200-L211) (verified)
  - *To reach the next level:* At least one extension type would need verification.
- **D L2:** Nothing third-party is enabled in the default task; an operator adds tool servers by editing the task YAML, with no display of what will run. — [agentverse/initialization.py:108](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/initialization.py#L108); [agentverse/tasks/tasksolving/brainstorming/config.yaml:111-112](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/tasks/tasksolving/brainstorming/config.yaml#L111-L112) (verified)
  - *To reach the next level:* Adding an extension should show the exact package, command and permissions.
- **B L2:** Tool servers are separate processes reached over HTTP and receive only tool arguments and cookies; inferred from the call sites and BMTools' documented HTTP wrapper behaviour, not from BMTools source. — [agentverse/environments/tasksolving_env/rules/executor/tool_using.py:315-321](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/tool_using.py#L315-L321) (inferred)
  - *To reach the next level:* Extensions are not sandboxed per extension with their own scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.05 (high)

API keys come from environment variables and there is no masking, redaction or secret handling anywhere in the code. The keys are not put into prompts, and full prompts are only written to the activity log when --debug is set, but tool inputs, tool outputs and execution results are logged unredacted at the default level. The largest gap is that every subprocess, including model-written code, inherits the long-lived provider keys.

- **S L0:** Secrets are plain environment variables with no masking or redaction on any path. — [agentverse/llms/openai.py:38](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/llms/openai.py#L38); searched `rg -n -i "redact|mask|SecretStr"` in `agentverse agentverse_command` → 0 hits (verified)
  - *To reach the next level:* No masking or redaction in even one path.
- **C L0:** No path is protected: logs, saved results and subprocess environments all get unfiltered data. — searched `rg -n "env="` in `agentverse/environments agentverse/agents` → 1 hits (only hit is httpx trust_env=True in tool_using.py:303; no subprocess call passes a scrubbed env); [agentverse/environments/tasksolving_env/rules/executor/tool_using.py:162-163](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/tool_using.py#L162-L163) (verified)
  - *To reach the next level:* Protect at least one path, e.g. scrub subprocess environments.
- **D L1:** No telemetry exists; default logging is INFO and omits prompts, but --debug writes full prompts to activity.log unredacted. — [agentverse_command/main_tasksolving_cli.py:26](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse_command/main_tasksolving_cli.py#L26); [agentverse/llms/openai.py:222](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/llms/openai.py#L222) (verified)
  - *To reach the next level:* Redaction is not available, so it can't be on by default.
- **B L0:** Long-lived provider keys are reachable by every subprocess, including model-written code. — [agentverse/llms/openai.py:38](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/llms/openai.py#L38); [agentverse/environments/tasksolving_env/rules/executor/code_test.py:18](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L18) (verified)
  - *To reach the next level:* Keys would need to be scoped or short-lived, or kept out of child processes.
- **Cap:** none

### C9 Audit & traceability — 0.30 (high)

AgentVerse writes a text activity log (activity.log, error.log) under the package directory, recording role assignments, plans, execution results and, for the tool-using executor, each tool name, input and observation. The records are unstructured, carry no actor or approver attribution, and do not log the exact command the code-test executor runs. The log sits outside the task working directory but is writable by the same user that runs model code.

- **S L1:** Unstructured formatted log lines of some actions. — [agentverse/logging.py:62-69](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/logging.py#L62-L69) (verified)
  - *To reach the next level:* No structured per-call record with arguments, status and timestamps for every tool call.
- **C L1:** The tool-using executor logs each call; the code-test executor's command and the simulation ToolAgent's tool calls are not logged. — [agentverse/environments/tasksolving_env/rules/executor/tool_using.py:162-163](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/tool_using.py#L162-L163); [agentverse/environments/tasksolving_env/rules/executor/code_test.py:45-47](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L45-L47) (verified)
  - *To reach the next level:* Log every built-in tool path.
- **D L2:** Logging is on by default and written to the package's logs directory, outside the task's cwd, but the same OS user that runs model code can edit it. — [agentverse/logging.py:41-42](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/logging.py#L41-L42) (verified)
  - *To reach the next level:* Logs would need to be written by a component model code can't control.
- **B L1:** Best-effort Python logging: handler errors are printed and execution continues. — [agentverse/logging.py:62-64](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/logging.py#L62-L64) (verified)
  - *To reach the next level:* Logging failures should surface and records be durable per action.
- **Cap:** none

### C10 Limits & kill switch — 0.38 (high)

Runs are bounded by a turn cap (10 by default, 3 for the default brainstorming task), a 10-call cap per tool-using executor round, a 10-second timeout on code-test execution and a 5-second timeout on in-process unit tests. Cost is tracked and printed but never enforced. The timeouts stop waiting rather than stopping work: killing the multiprocessing worker leaves the shell's python child running, and the timed-out thread keeps executing. The simulation tool agent loops with while True until the model finishes and catches BaseException, which also swallows Ctrl+C during a call.

- **S L2:** An iteration cap plus per-execution timeouts are enforced in code; there is no wall-clock or cost cap. — [agentverse/environments/tasksolving_env/basic.py:136](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/basic.py#L136); [agentverse/environments/tasksolving_env/rules/executor/code_test.py:49-51](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L49-L51); [agentverse/environments/tasksolving_env/rules/executor/tool_using.py:112](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/tool_using.py#L112) (verified)
  - *To reach the next level:* No session wall-clock or token/cost cap; spend is only reported.
- **C L1:** The top-level turn loop and tool executors are bounded, but the simulation ToolAgent's inner while True loop has no step cap, so one agent step can run indefinitely beneath the turn limit. — [agentverse/agents/simulation_agent/tool.py:39](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/agents/simulation_agent/tool.py#L39); [agentverse/environments/tasksolving_env/rules/executor/tool_using.py:323-327](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/tool_using.py#L323-L327) (verified)
  - *To reach the next level:* Every inner loop (ToolAgent) would need a step cap so tool timeouts plus the top-level loop actually bound the run.
- **D L2:** Defaults are sensible (max_turn 10, brainstorming 3) and operator-configurable in task YAML. — [agentverse/environments/tasksolving_env/basic.py:26](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/basic.py#L26); [agentverse/tasks/tasksolving/brainstorming/config.yaml:2](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/tasks/tasksolving/brainstorming/config.yaml#L2) (verified)
  - *To reach the next level:* Hard ceilings or protection against the model's plan raising the number of parallel tool agents are absent.
- **B L1:** Stopping leaves work running: timed-out shell children and threads are not killed, and BaseException handlers swallow KeyboardInterrupt in the tool agent. — [agentverse/environments/tasksolving_env/rules/executor/code_test.py:18](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/tasksolving_env/rules/executor/code_test.py#L18); [agentverse/environments/simulation_env/rules/selector/code_api.py:52-57](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/environments/simulation_env/rules/selector/code_api.py#L52-L57); [agentverse/agents/simulation_agent/tool.py:53](https://github.com/openbmb/agentverse/blob/f90c4bd9680fdd3bcff8c52c9170911a59b23478/agentverse/agents/simulation_agent/tool.py#L53) (verified)
  - *To reach the next level:* Stop should kill the process group and cancel pending calls.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web pages via WebEnv_browse_website/search (executor/tool_using.py:329-353) · [B] sensitive data/systems: OPENAI_API_KEY/AZURE keys in env (llms/openai.py:38-40), inherited by executed code (executor/code_test.py:18) · [C] state change / egress: host shell (code_test.py:18) and arbitrary ToolServer calls incl. root shell (tool_using.py:318-327) · Same default session? Yes

## Highest-impact improvements
1. Run code-test and sde_team execution in a locked-down container (non-root, no network, workspace-only mount) with a scrubbed environment, failing closed if unavailable. — C4 S L0→L3, +0.225 before caps (Playbook 3)
2. Pass env={} (or an explicit allowlist) to every subprocess so model-run code never inherits provider keys. — C8 C L0→L1, +0.075 before caps (Playbook 4)
3. Reject tool calls whose name is not in the configured tool_names list and confine code-test file_path to a temp directory via realpath containment. — C3 S L0→L2, +0.150 before caps (Playbook 3)
4. Add a default-on per-call approval prompt showing the exact command or tool arguments before shell, file-write and tool-server calls. — C2 S L0→L3, +0.225 before caps (Playbook 5)
5. Avoid evaluating model output in-process, and kill the process group on code-test timeout. — C10 B L1→L2, +0.050 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of commit f90c4bd only; nothing was executed, installed, or probed.
- Framework scored by its defaults and shipped task configs: absent primitives score L0 even where a developer could add them.
- The external XAgent ToolServer and BMTools were not part of this repository and were not examined; their isolation and behaviour are not credited.
- Model behaviour is out of scope; only code-level controls are scored.
- No reviewer-directed prompt injection found in the repo.
- The ui/ (Phaser front-end) and pokemon_server.py demo were only skimmed.
