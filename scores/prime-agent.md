# Defense-in-depth score: Prime Agent

**Repo:** https://github.com/PrimeIntellect-ai/prime-agent · **Commit:** `41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f` · **Reviewed:** 2026-10-05
**What it is:** Open-source coding and research agent built around a persistent Python REPL, recursive sub-agents and a daemon for long-running sessions.
**Category:** Coding
**Scored configuration:** Interactive `prime-agent` CLI backed by its daemon, no flags, fresh install: the ipython REPL as the only model tool, autonomous mode off, no MCP servers or packages configured by the user.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 1.5 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | C2-POWERBYPASS | **0.00** | High |
| C3 | Tool & action scoping | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L1 | L1 | L1 | 0.17 | C6-REPOCONFIG | **0.17** | High |
| C7 | Third-party extensions | L1 | L1 | L0 | L0 | 0.15 | C7-RCELOAD | **0.15** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | none | **0.20** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L3 | 0.55 | none | **0.55** | High |
| C10 | Limits & kill switch | L2 | L1 | L2 | L1 | 0.38 | G1 | **0.38** (alt) | High |


Prime Agent runs model-written Python and shell commands directly on your machine with your full user permissions and environment, with no approval step and no sandbox; its README says as much. Anything it reads, including files in the repository it is pointed at, can steer it into leaking credentials or making irreversible changes. A repository can also add MCP servers, packages and executable skills through its project settings file without any trust prompt. Use it only on trusted repositories, ideally inside an external sandbox.

## Critical gaps
- Model-written code runs with the user's full authority and every credential in the environment. (ASI03, T3, LLM06; C1). Evidence: [crates/pa-core/src/kernel/manager/startup.rs:173](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/kernel/manager/startup.rs#L173); [crates/pa-core/src/kernel/manager/startup.rs:194](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/kernel/manager/startup.rs#L194)
- Every tool call, including arbitrary shell commands through the Python REPL, runs without human approval. (ASI09, ASI02, T10, LLM06; C2). Evidence: [crates/pa-core/src/tools/ipython.rs:365](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/tools/ipython.rs#L365); [crates/pa-daemon/src/agent_engine/lifecycle.rs:1160-1162](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-daemon/src/agent_engine/lifecycle.rs#L1160-L1162)
- Model-generated code runs unsandboxed on the host with the user's full environment; the project documents that it is not a security sandbox. (ASI05, T11, LLM05; C4). Evidence: [README.md:79](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/README.md#L79); [crates/pa-core/src/kernel/manager/startup.rs:173](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/kernel/manager/startup.rs#L173)
- A hijacked session can leak credentials and take irreversible actions with no human in the loop. (ASI01, T6, LLM01; C5). Evidence: [crates/pa-core/src/kernel/manager/startup.rs:173](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/kernel/manager/startup.rs#L173); [crates/pa-core/src/tools/ipython.rs:365](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/tools/ipython.rs#L365)
- Files in the working directory can add MCP servers, packages and executable skills without any trust prompt. (ASI06, T1, LLM04; C6). Evidence: [crates/pa-core/src/settings/storage.rs:79](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/settings/storage.rs#L79); [crates/pa-core/src/settings/manager.rs:56](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/settings/manager.rs#L56); [crates/pa-core/src/packages/resolve/manager.rs:153](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/packages/resolve/manager.rs#L153)
- Project-listed packages are installed automatically at session start, and skills run inside the kernel with every credential. (ASI04, T17, LLM03; C7). Evidence: [crates/pa-core/src/packages/resolve/manager.rs:131](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/packages/resolve/manager.rs#L131); [crates/pa-core/src/packages/resolve/manager.rs:153](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/packages/resolve/manager.rs#L153); [crates/pa-core/src/kernel/bootstrap/venv.rs:220](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/kernel/bootstrap/venv.rs#L220)

## Criterion details

### C1 Identity & least privilege: 0.00 (high confidence)

Prime Agent runs as the logged-in user with that user's full authority. The Python kernel that executes all model code is started with a copy of the entire host environment, and every shell command it runs inherits it again, so provider API keys and any cloud or git credentials in the environment are available to model-written code. The stored credential file in the agent directory is readable by the same process. There is no per-tool identity or authorization check; a hijacked session can do anything the user can.

- **S L0:** The agent uses the operator's ambient authority: the kernel inherits the full process environment and runs as the user. Evidence: [crates/pa-core/src/kernel/manager/startup.rs:173](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/kernel/manager/startup.rs#L173); [crates/pa-core/src/kernel/manager/startup.rs:194](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/kernel/manager/startup.rs#L194); [README.md:79](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/README.md#L79) (verified)
  - *To reach the next level:* No narrowing of the user's authority: no scoped or per-tool credentials and no authorization gate before actions.
- **C L0:** Every action goes through model-written Python and shell in the kernel, which receives the full environment with no authorization layer. Evidence: [prime-agent-runtime/src/rlm/bash.py:1051-1063](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/prime-agent-runtime/src/rlm/bash.py#L1051-L1063); [crates/pa-core/src/tools/ipython.rs:365](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/tools/ipython.rs#L365) (verified)
  - *To reach the next level:* No authorization layer on any tool path; subprocesses inherit the full environment.
- **D L0:** The default install runs with everything the user can reach; there is no restricted default. Evidence: [crates/pa-daemon/src/agent_engine/lifecycle.rs:1160-1162](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-daemon/src/agent_engine/lifecycle.rs#L1160-L1162); [crates/pa-core/src/kernel/manager/startup.rs:173](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/kernel/manager/startup.rs#L173) (verified)
  - *To reach the next level:* No narrower default role or credential set exists.
- **B L0:** A hijacked session holds the user's whole account on the machine plus every credential in the environment and the stored auth file. Evidence: [crates/pa-core/src/kernel/manager/startup.rs:173](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/kernel/manager/startup.rs#L173); [crates/pa-core/src/auth/storage.rs:1](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/auth/storage.rs#L1) (verified)
  - *To reach the next level:* Blast radius is not reduced: credentials are long-lived and cover every service the user is signed into.
- **Cap:** none

### C2 Approval gates: 0.00 (high confidence)

There is no approval step. The only model tool is a persistent Python REPL that can run shell commands, edit files, call MCP servers and spawn sub-agents, and every call runs immediately. A dirty-tree guard for destructive git commands exists in the codebase but is attached to a shell tool that the live agent does not expose. The README tells users to work in a disposable clone, which is the only protection against unwanted changes.

- **S L0:** No human approval of any kind is implemented for tool calls. Evidence: searched `rg -n -i -e 'requires_approval|approval_policy|ask_permission|permission_mode|confirm_tool|auto_approve'` in `crates/pa-core/src crates/pa-agent/src crates/pa-daemon/src prime-agent-runtime/src` → 0 hits (no tool-approval layer exists); [crates/pa-daemon/src/agent_engine/lifecycle.rs:1160-1162](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-daemon/src/agent_engine/lifecycle.rs#L1160-L1162) (verified)
  - *To reach the next level:* No per-call approval showing the exact code or command.
- **C L0:** The most powerful path, arbitrary Python and shell in the kernel, is the only tool and runs ungated. Evidence: [crates/pa-core/src/tools/ipython.rs:365](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/tools/ipython.rs#L365); [crates/pa-daemon/src/agent_engine/lifecycle.rs:1160-1162](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-daemon/src/agent_engine/lifecycle.rs#L1160-L1162) (verified)
  - *To reach the next level:* The code-execution tool is not behind any gate.
- **D L0:** Approval does not exist, so it is effectively off by default. Evidence: searched `rg -n -i -e 'requires_approval|approval_policy|ask_permission|permission_mode|confirm_tool|auto_approve'` in `crates/pa-core/src crates/pa-agent/src crates/pa-daemon/src prime-agent-runtime/src` → 0 hits (no tool-approval layer exists) (verified)
  - *To reach the next level:* No approval mode at all, let alone one on by default.
- **B L0:** A wrong action can delete files, push code or call external services with no undo; the git guard is not on the live tool path. Evidence: [crates/pa-core/src/tools/golden_replay.rs:623](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/tools/golden_replay.rs#L623); [README.md:79](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/README.md#L79) (verified)
  - *To reach the next level:* No checkpoint or rollback for working-tree changes made by the agent.
- **Cap:** C2-POWERBYPASS: The single most powerful action path, the ipython tool with shell access, executes without any gate in the default configuration.

### C3 Tool & action scoping: 0.00 (high confidence)

The agent's only tool takes a free-form Python string. File paths, URLs, shell commands and package installs are all expressed as code, so there is nothing for an argument check to inspect. The tool set cannot be narrowed: the old tool-selection flags were removed and the daemon always configures the REPL as the only tool.

- **S L0:** The tool is raw passthrough of arbitrary Python, including shell via bash(). Evidence: [crates/pa-core/src/tools/ipython.rs:365](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/tools/ipython.rs#L365) (verified)
  - *To reach the next level:* No argument validation; no narrow tools replacing the general interpreter.
- **C L0:** No tool validates inputs. Evidence: [crates/pa-daemon/src/agent_engine/lifecycle.rs:1160-1162](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-daemon/src/agent_engine/lifecycle.rs#L1160-L1162) (verified)
  - *To reach the next level:* No validation layer on any tool path.
- **D L0:** Exec, write and network capability are all enabled by default through the REPL. Evidence: [crates/pa-daemon/src/agent_engine/lifecycle.rs:1160-1162](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-daemon/src/agent_engine/lifecycle.rs#L1160-L1162); [crates/pa-core/src/tools/ipython.rs:365](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/tools/ipython.rs#L365) (verified)
  - *To reach the next level:* No read-only default tool set or way to disable the exec tool.
- **B L0:** A misused tool can act on the whole machine and any network host. Evidence: [crates/pa-core/src/kernel/manager/startup.rs:173](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/kernel/manager/startup.rs#L173); [prime-agent-runtime/src/rlm/bash.py:1051-1063](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/prime-agent-runtime/src/rlm/bash.py#L1051-L1063) (verified)
  - *To reach the next level:* No workspace scoping or quantity bounds on what tools can touch.
- **Cap:** none

### C4 Code-execution isolation: 0.00 (high confidence)

Model-written Python and shell commands run directly on the host as the user. The README states plainly that the worker and kernel processes are not a security sandbox and advises running untrusted work elsewhere. No container, OS sandbox profile or VM is available as an option, and the kernel receives the full host environment, so code it runs can read credentials and reach the network.

- **S L0:** Execution is an ordinary same-user subprocess; no isolation primitive exists. Evidence: searched `rg -n -S -e 'landlock|seatbelt|sandbox-exec|bwrap|bubblewrap|nsjail|firejail|gvisor|runsc|firecracker'` in `crates/pa-core/src crates/pa-daemon/src prime-agent-runtime/src` → 0 hits (no OS or VM isolation primitive anywhere in the host or the Python runtime); [README.md:79](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/README.md#L79) (verified)
  - *To reach the next level:* No OS-level separation such as a container, dedicated user or sandbox profile.
- **C L0:** No execution path is sandboxed. Evidence: [crates/pa-core/src/kernel/manager/startup.rs:194](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/kernel/manager/startup.rs#L194); [prime-agent-runtime/src/rlm/bash.py:1051-1063](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/prime-agent-runtime/src/rlm/bash.py#L1051-L1063) (verified)
  - *To reach the next level:* The main exec tool is not sandboxed.
- **D L0:** No sandbox is on by default because none exists. Evidence: [README.md:79](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/README.md#L79) (verified)
  - *To reach the next level:* No sandbox mode to enable by default.
- **B L0:** Executed code reaches the home directory, credentials in the environment and the network. Evidence: [crates/pa-core/src/kernel/manager/startup.rs:173](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/kernel/manager/startup.rs#L173); [prime-agent-runtime/src/rlm/bash.py:1051-1063](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/prime-agent-runtime/src/rlm/bash.py#L1051-L1063) (verified)
  - *To reach the next level:* Executed code has host-equivalent reach with credentials present.
- **Cap:** none

### C5 Untrusted input blast radius: 0.00 (high confidence)

Content the agent reads, from repository files, web search, MCP servers or messages sent by other running agents, enters the model context with the same standing as the user's instructions. Nothing marks it as untrusted or limits what the agent can do after reading it. The same session holds the user's credentials and can run any shell command or network call, so a successful injection can both leak secrets and take irreversible actions without a human involved.

- **S L0:** No structural limit on a hijacked agent; no provenance, taint tracking or rule-of-two enforcement. Evidence: searched `rg -n -i -e 'untrusted|prompt.?injection|quarantin'` in `crates/pa-core/src/session_engine crates/pa-agent/src prime-agent-runtime/src` → 0 hits (tool results and other agents' messages carry no provenance or taint) (verified)
  - *To reach the next level:* No human approval of egress or state change after untrusted content is read.
- **C L0:** Untrusted sources are not distinguished from user input. Evidence: searched `rg -n -i -e 'untrusted|prompt.?injection|quarantin'` in `crates/pa-core/src/session_engine crates/pa-agent/src prime-agent-runtime/src` → 0 hits (tool results and other agents' messages carry no provenance or taint); [crates/pa-core/src/tools/ipython.rs:365](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/tools/ipython.rs#L365) (verified)
  - *To reach the next level:* No source is handled as untrusted.
- **D L0:** No control exists to be on by default. Evidence: searched `rg -n -i -e 'untrusted|prompt.?injection|quarantin'` in `crates/pa-core/src/session_engine crates/pa-agent/src prime-agent-runtime/src` → 0 hits (tool results and other agents' messages carry no provenance or taint) (verified)
  - *To reach the next level:* No untrusted-input control at all.
- **B L0:** A hijacked session can exfiltrate credentials and files and take irreversible actions unattended. Evidence: [crates/pa-core/src/kernel/manager/startup.rs:173](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/kernel/manager/startup.rs#L173); [crates/pa-core/src/tools/ipython.rs:365](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/tools/ipython.rs#L365) (verified)
  - *To reach the next level:* Leak and irreversible action both remain possible without a human.
- **Cap:** C5-WORSTCASE: Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity: 0.17 (high confidence)

Several things persist into future behaviour without a trust decision. The project-level settings file in the working directory is merged over the user's settings and can declare MCP servers, packages, skills and shell settings; project packages are installed and project skills are installed into the kernel when a session starts; AGENTS.md and CLAUDE.md files are loaded silently. The continual-harness memory is better contained: automatic refinement is gated by a model review, always writes the session-local store, and keeps a history that supports rollback, but the model can also write global state directly through its shell.

- **S L0:** Repository files can add MCP servers, packages and executable skills with no prompt. Evidence: [crates/pa-core/src/settings/storage.rs:79](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/settings/storage.rs#L79); [crates/pa-core/src/settings/manager.rs:56](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/settings/manager.rs#L56); searched `rg -n -i -e 'workspace.?trust|trust.?folder|trusted.?project|project.?trust'` in `crates/pa-core/src crates/pa-daemon/src crates/pa-cli/src` → 0 hits (no workspace-trust decision exists) (verified)
  - *To reach the next level:* Security-relevant project config is not behind an explicit workspace-trust decision.
- **C L1:** Only the automatic refinement path is constrained (session-local scope in code); instruction files and project config are not. Evidence: [crates/pa-core/src/session_engine/refine.rs:631](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/session_engine/refine.rs#L631); [crates/pa-core/src/resources/mod.rs:17](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/resources/mod.rs#L17) (verified)
  - *To reach the next level:* Auto-loaded instruction files and project settings are not controlled.
- **D L1:** Automatic refinement defaults to the per-session store, enforced in code, but the level is limited by the weak load control. Evidence: [crates/pa-core/src/session_engine/refine.rs:631](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/session_engine/refine.rs#L631); [crates/pa-core/src/session_engine/refine.rs:49](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/session_engine/refine.rs#L49) (verified)
  - *To reach the next level:* Isolation is undermined because any repository can inject persistent configuration and the model can write global harness state through its shell.
- **B L1:** Poisoned project config or global harness state persists across the user's sessions and can trigger tool use. Evidence: [crates/pa-core/src/settings/storage.rs:79](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/settings/storage.rs#L79); [crates/pa-core/src/skills/loader.rs:128](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/skills/loader.rs#L128) (verified)
  - *To reach the next level:* Persisted entries can drive tool use in later sessions without review.
- **Cap:** C6-REPOCONFIG: Project settings in the working directory are merged without a trust decision and can add MCP servers, packages and executable skills.

### C7 Third-party extensions: 0.15 (high confidence)

Extensions come from MCP server entries, npm or git packages and Python skill packages. Packages can be pinned and catalog MCP services carry reviewed metadata and a fixed endpoint, but user-declared servers and unpinned packages run whatever is current. Packages listed in a project's settings file are installed automatically when a session starts, and Python skills are installed into and imported by the kernel, so they run inside it with the full environment. Stdio MCP servers do get a reduced environment.

- **S L1:** Sources are chosen in settings and pinning is supported but not required; there is no integrity check. Evidence: [crates/pa-core/src/packages/resolve/manager.rs:131](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/packages/resolve/manager.rs#L131); [crates/pa-core/src/packages/resolve/manager.rs:153](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/packages/resolve/manager.rs#L153) (verified)
  - *To reach the next level:* Versions are not pinned by default and nothing verifies a hash or signature.
- **C L1:** Only catalog MCP services have reviewed metadata and endpoint binding; packages, skills and user-declared servers are unverified. Evidence: [crates/pa-core/src/mcp/local_catalog.rs:8-11](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/mcp/local_catalog.rs#L8-L11) (verified)
  - *To reach the next level:* Most extension types (packages, skills, user MCP servers) get no verification.
- **D L0:** A workspace settings file can add packages that are installed automatically at session start. Evidence: [crates/pa-core/src/packages/resolve/manager.rs:131](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/packages/resolve/manager.rs#L131); [crates/pa-core/src/packages/resolve/manager.rs:153](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/packages/resolve/manager.rs#L153); [crates/pa-core/src/resources/resolution.rs:182](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/resources/resolution.rs#L182) (verified)
  - *To reach the next level:* Extensions can be added from the workspace with no consent step.
- **B L0:** Python skills and package skills run inside the kernel process with the agent's full environment. Evidence: [crates/pa-core/src/kernel/bootstrap/venv.rs:220](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/kernel/bootstrap/venv.rs#L220); [crates/pa-core/src/kernel/manager/startup.rs:173](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/kernel/manager/startup.rs#L173) (verified)
  - *To reach the next level:* Extensions are not confined: skills run in-process with every credential.
- **Cap:** C7-RCELOAD: By default, package sources listed in a project's settings file are fetched and installed at session start without consent.

### C8 Secrets & sensitive-data protection: 0.20 (high confidence)

Provider credentials come from environment variables or a plaintext auth file kept with owner-only permissions. MCP diagnostics scrub configured values and stdio MCP servers get a reduced environment, but the kernel that runs all model code receives the full environment, and session transcripts are written without redaction. Product telemetry is on by default; its schema is limited to primitive properties with no prompt or tool content.

- **S L1:** Secrets come from env vars and an owner-only plaintext file; masking exists only for MCP diagnostics. Evidence: [crates/pa-core/src/auth/storage.rs:1](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/auth/storage.rs#L1); [prime-agent-runtime/src/rlm/mcp.py:1037](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/prime-agent-runtime/src/rlm/mcp.py#L1037) (verified)
  - *To reach the next level:* No redaction before logs, transcripts or model-bound messages on main paths.
- **C L1:** Protection covers MCP diagnostics and the stdio MCP environment; the kernel environment, transcripts and model context are unprotected. Evidence: [prime-agent-runtime/src/rlm/mcp.py:981-982](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/prime-agent-runtime/src/rlm/mcp.py#L981-L982); searched `rg -n -i -e 'redact|mask_secret|scrub'` in `crates/pa-core/src/session crates/pa-agent/src` → 4 hits (all four hits are model 'redacted thinking' flags, not secret redaction; transcripts are written unredacted) (verified)
  - *To reach the next level:* Transcripts and the kernel subprocess environment are not protected.
- **D L1:** Telemetry is on by default and content-free by schema. Evidence: [crates/pa-core/src/settings/manager.rs:949](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/settings/manager.rs#L949); [crates/pa-telemetry/src/lib.rs:9](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-telemetry/src/lib.rs#L9) (verified)
  - *To reach the next level:* Telemetry is not opt-in.
- **B L0:** Long-lived provider keys and OAuth tokens are reachable by every model-run subprocess. Evidence: [crates/pa-core/src/kernel/manager/startup.rs:173](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/kernel/manager/startup.rs#L173); [prime-agent-runtime/src/rlm/bash.py:1051-1063](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/prime-agent-runtime/src/rlm/bash.py#L1051-L1063) (verified)
  - *To reach the next level:* Keys reachable by the model are long-lived and broadly scoped.
- **Cap:** none

### C9 Audit & traceability: 0.55 (high confidence)

Each session is recorded as a structured JSONL transcript under the agent directory, including the Python code of every tool call, its output and bash command details, and each row is synced to disk as it is written so a session can be replayed. Sub-agents keep their own transcripts. The record is outside the working directory but in a location the agent's own shell can edit, there is no attribution of who approved what (there are no approvals), and nothing is hash-chained or shipped off the machine.

- **S L2:** Structured local transcript of every tool call with arguments, results and timestamps. Evidence: [crates/pa-daemon/src/paths.rs:60-68](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-daemon/src/paths.rs#L60-L68); [crates/pa-core/src/session/window_cache.rs:203](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/session/window_cache.rs#L203) (verified)
  - *To reach the next level:* No actor or approver attribution and no correlation across sub-agents in a single record.
- **C L2:** All built-in tool calls are recorded; shell and MCP actions appear only as the Python code that invoked them. Evidence: [crates/pa-core/src/tools/ipython.rs:365](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/tools/ipython.rs#L365); [crates/pa-daemon/src/paths.rs:60-68](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-daemon/src/paths.rs#L60-L68) (verified)
  - *To reach the next level:* MCP calls and shell commands inside a cell are not recorded as separate attributed events.
- **D L2:** On by default and stored in the agent directory outside the workspace. Evidence: [crates/pa-daemon/src/paths.rs:60-68](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-daemon/src/paths.rs#L60-L68) (verified)
  - *To reach the next level:* The agent's own process can still alter or delete its transcripts.
- **B L3:** Rows are appended and synced per entry, and append failures are returned as errors; the transcript replays on resume. Evidence: [crates/pa-core/src/session/window_cache.rs:203](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/session/window_cache.rs#L203) (verified)
  - *To reach the next level:* Actions are not blocked when their record cannot be written (not fail-closed).
- **Cap:** none

### C10 Limits & kill switch: 0.38 (high confidence)

In the default interactive mode the turn loop has no step, time or cost limit, and individual code cells have no default timeout. Interrupting stops an awaited shell command's process group, but background handles, schedules, heartbeats and daemon-backed sessions are designed to keep running after the terminal detaches. Sub-agent nesting is limited to depth 2 by default. The opt-in autonomous mode adds real budgets (12 turns, 80,000 tokens and 30 minutes by default), enforced in code.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** No step, time or cost limit applies to an interactive session. Evidence: [crates/pa-agent/src/agent_loop/run.rs:51](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-agent/src/agent_loop/run.rs#L51); searched `rg -n -e 'max_turns|max_steps|max_iterations'` in `crates/pa-agent/src` → 0 hits (the agent turn loop has no step cap) (verified)
    - *To reach the next level:* No iteration cap or per-call timeout in the default loop.
  - **C L0:** No budget applies to the loop or tools; only delegation depth is capped. Evidence: [crates/pa-daemon/src/rlm_children.rs:42](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-daemon/src/rlm_children.rs#L42); searched `rg -n -e 'max_turns|max_steps|max_iterations'` in `crates/pa-agent/src` → 0 hits (the agent turn loop has no step cap) (verified)
    - *To reach the next level:* Limits do not apply to the top-level loop.
  - **D L0:** Unlimited by default. Evidence: searched `rg -n -e 'max_turns|max_steps|max_iterations'` in `crates/pa-agent/src` → 0 hits (the agent turn loop has no step cap) (verified)
    - *To reach the next level:* No default budget for interactive sessions.
  - **B L0:** A runaway session can spend and act indefinitely, and background work survives a stop. Evidence: [prime-agent-runtime/src/rlm/bash.py:965-967](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/prime-agent-runtime/src/rlm/bash.py#L965-L967); [crates/pa-agent/src/agent_loop/run.rs:51](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-agent/src/agent_loop/run.rs#L51) (verified)
    - *To reach the next level:* No spend or time ceiling; stopping can leave background processes and schedules running.
- **opt-in autonomous mode budgets** (alt; raw 0.38, cap G1 → 0.38) ← counted
  - **S L2:** Turn, token and wall-clock budgets are enforced in code for autonomous continuations. Evidence: [crates/pa-core/src/autonomous/mod.rs:26-29](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/autonomous/mod.rs#L26-L29); [crates/pa-core/src/autonomous/mod.rs:396-405](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/autonomous/mod.rs#L396-L405) (verified)
    - *To reach the next level:* No rate limit on side-effecting tools and no repeated-action breaker.
  - **C L1:** Budgets cover the top-level autonomous loop; sub-agent consumption against the same budget was not confirmed. Evidence: [crates/pa-core/src/autonomous/mod.rs:396-405](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/autonomous/mod.rs#L396-L405); [crates/pa-daemon/src/rlm_children.rs:42](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-daemon/src/rlm_children.rs#L42) (verified)
    - *To reach the next level:* No confirmed shared budget for sub-agents and background tasks.
  - **D L2:** Sensible defaults (12 turns, 80k tokens, 30 minutes) that the operator can change, including to unlimited. Evidence: [crates/pa-core/src/autonomous/mod.rs:26-29](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/crates/pa-core/src/autonomous/mod.rs#L26-L29) (verified)
    - *To reach the next level:* No hard ceiling configuration cannot exceed.
  - **B L1:** Ceilings are moderate, but stopping can leave background shell handles and scheduled work running. Evidence: [prime-agent-runtime/src/rlm/bash.py:965-967](https://github.com/PrimeIntellect-ai/prime-agent/blob/41e1f41c072e8f31ac6a79ad1b7fe19920c6d61f/prime-agent-runtime/src/rlm/bash.py#L965-L967) (verified)
    - *To reach the next level:* Stopping does not end background processes and scheduled work.
- **Cap:** G1: Opt-in mechanism: off in the scored default configuration.

## Rule-of-Two check
[A] untrusted input: Repository files, AGENTS.md, web search, MCP results and other agents' messages enter context unmarked (crates/pa-core/src/resources/mod.rs:17) · [B] sensitive data/systems: Kernel inherits the full environment and can read the stored auth file (crates/pa-core/src/kernel/manager/startup.rs:173) · [C] state change / egress: Arbitrary Python and shell with network access (crates/pa-core/src/tools/ipython.rs:365) · Same default session? Yes

## Highest-impact improvements
1. Require an explicit workspace-trust decision before loading project settings that add MCP servers, packages, skills or shell settings. (C6 S L0→L3, +0.225 before caps; Playbook 2)
2. Do not auto-install packages listed in project settings; show the exact source and ask first. (C7 D L0→L3, +0.150 before caps; Playbook 3)
3. Add a per-call approval mode for kernel cells that run shell commands or touch files outside the workspace, on by default. (C2 S L0→L3, +0.225 before caps; Playbook 5)
4. Start the kernel with a scrubbed environment and pass provider credentials only to the host process. (C1 C L0→L1, +0.075 before caps; Playbook 4)
5. Apply default step, token and wall-clock budgets to interactive sessions, not only autonomous mode. (C10 S L0→L2, +0.150 before caps; Playbook 3 step 3)

## Re-audit log
- C9 B: L2 → L3. Per-row sync_data on every transcript append (window_cache.rs:203) and resumable JSONL meet 'durable per action; replayable'.
- C7 B: L2 → L0. Stdio MCP servers get a scrubbed environment, but Python skills and package skills are installed into and imported by the kernel, which holds the full environment.
- C2 B: L1 → L0. The destructive-git guard is wired only to a bash tool definition referenced from golden replay, not to the live tool set (ipython only).

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the daemon-backed interactive CLI; headless print mode, ACP mode and the factory workflow engine were read only where they share code with it.
- Whether autonomous-mode token budgets include sub-agent usage was not confirmed (C10 alt C rated conservatively).
- The Python runtime's third-party dependencies (MCP SDK, IPython) were not reviewed.
- No reviewer-steering text was found in the repository's markdown files.
