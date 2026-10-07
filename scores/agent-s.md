# Defense-in-Depth Score: Agent S

**Repo:** https://github.com/simular-ai/Agent-S · **Commit:** `3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693` · **Reviewed:** 2026-10-03
**What it is:** Computer-use agent operating a real GUI via mouse/keyboard
**Category:** AI Assistants
**Scored configuration:** The `agent_s` console script (Agent S3, gui_agents/s3/cli_app.py) with only the required model/grounding arguments; --enable_local_env off, reflection on; equivalently the AgentS3/OSWorldACI SDK classes with default arguments and env=None.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents opt-in · external communication yes

## Score: 1.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | C2-POWERBYPASS | **0.00** | High |
| C3 | Tool & action scoping | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | C7-RCELOAD | **0.00** | High |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C9 | Audit & traceability | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |

Controls where a risk surface exists: 0.57 / 9.0 (6%); 1 criterion scored SA (surface absent).

Agent S takes over your real mouse, keyboard and screen and executes the model's chosen action as Python inside its own process, with no sandbox and no human approval: the permission dialog in the CLI is never called. Anything on screen, such as a web page or email, can steer the model into running arbitrary code with your full user rights, API keys and a sudo password from its prompt. Only run it inside a disposable VM with no personal accounts or credentials.

## Critical gaps
- Model-generated code runs in-process with the user's full ambient authority, including API keys in os.environ and a sudo password supplied in the prompt. (ASI03, T3; C1) — [gui_agents/s3/utils/common_utils.py:31](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/common_utils.py#L31); [gui_agents/s3/memory/procedural_memory.py:110](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/memory/procedural_memory.py#L110)
- The approval dialog is dead code: the CLI comments 'Ask for permission before executing' and then exec()s the model's action without calling it. (ASI09, ASI02; C2) — [gui_agents/s3/cli_app.py:214-215](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L214-L215); [gui_agents/s3/cli_app.py:133](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L133)
- Model-generated Python is eval()'d and exec()'d in the agent process on the user's host with no sandbox, in the default configuration. (ASI05, T11; C4) — [gui_agents/s3/utils/common_utils.py:31](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/common_utils.py#L31); [gui_agents/s3/cli_app.py:214-215](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L214-L215)
- On-screen content from any website or document can hijack the model, whose output is then executed as arbitrary Python without approval, enabling unattended exfiltration and irreversible actions. (ASI01, LLM01, T6; C5) — [gui_agents/s3/cli_app.py:165](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L165); [gui_agents/s3/utils/common_utils.py:31](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/common_utils.py#L31); [gui_agents/s3/cli_app.py:214-215](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L214-L215)
- The default type action auto-installs an unpinned PyPI package (and runs sudo apt-get with a hardcoded password) and imports it into the agent process without consent. (ASI04, T17; C7) — [gui_agents/s3/agents/grounding.py:434-435](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/grounding.py#L434-L435)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

Agent S runs with the full authority of the logged-in user and narrows nothing. The model's chosen action is evaluated as raw Python inside the agent process, so it can reach every file, environment variable (including the model API keys), browser session and network destination the user can. The worker prompt even hands the model a sudo password and the generated typing code runs sudo, so a hijacked agent is the user, and possibly root on a machine that reuses that benchmark password.

- **S L0:** No dedicated identity: model-generated code runs via eval in the agent process with the OS user's ambient authority, and API keys come from the process environment. — [gui_agents/s3/utils/common_utils.py:31](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/common_utils.py#L31); [gui_agents/s3/core/engine.py:43](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/core/engine.py#L43) (verified)
  - *To reach the next level:* Use a dedicated low-privilege OS user or scoped credentials per capability instead of the operator's ambient authority.
- **C L0:** Every action, including the opt-in bash/python code agent subprocesses, inherits the full parent environment; no authorization check exists on any path. — [gui_agents/s3/utils/local_env.py:15-19](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/local_env.py#L15-L19); searched `rg -n 'env='` in `gui_agents/s3` → 1 hits (Only hit is cli_app.py:360 passing env=local_env to OSWorldACI; no subprocess receives a scrubbed environment.) (verified)
  - *To reach the next level:* Pass a scrubbed env to subprocesses and put every action behind one authorization check.
- **D L0:** The default install runs as the user and the shipped prompt supplies a sudo password for privilege elevation. — [gui_agents/s3/memory/procedural_memory.py:110](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/memory/procedural_memory.py#L110); [gui_agents/s3/agents/grounding.py:434](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/grounding.py#L434) (verified)
  - *To reach the next level:* Remove the sudo credential from prompts and default to a non-privileged, read-mostly mode.
- **B L0:** A hijacked agent holds the user's entire desktop session: all files, logged-in web accounts seen through the GUI, cloud/SSH credentials on disk, and the model API keys. — [gui_agents/s3/utils/common_utils.py:31](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/common_utils.py#L31); [gui_agents/s3/cli_app.py:165](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L165) (verified)
  - *To reach the next level:* Confine the agent to a dedicated VM or account with no access to the operator's personal sessions or credentials.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no human approval anywhere. The CLI contains a permission-dialog function and a comment saying it asks for permission, but the function is never called, and the next line executes the model's action directly. Every click, keystroke, file operation or command the model chooses runs immediately, including irreversible ones such as sending email or deleting files through the GUI.

- **S L0:** No approval step: the comment says 'Ask for permission' but the next line is exec(code[0]); show_permission_dialog is dead code. — [gui_agents/s3/cli_app.py:214-215](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L214-L215); searched `rg -n 'show_permission_dialog'` in `gui_agents/s3` → 1 hits (Only the definition at cli_app.py:133; the function is never called.) (verified)
  - *To reach the next level:* Call a per-action approval that shows the exact code to be executed before exec.
- **C L0:** The most powerful path, raw eval of model output, runs before and independent of any gate, even during format validation. — [gui_agents/s3/utils/common_utils.py:31](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/common_utils.py#L31); [gui_agents/s3/utils/formatters.py:30-35](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/formatters.py#L30-L35); searched `rg -n -i 'approv|confirm|permission'` in `gui_agents/s3` → 5 hits (All five hits are the unused show_permission_dialog body (133,134,137,142) and the comment at cli_app.py:214; no gate is executed.) (verified)
  - *To reach the next level:* Route every action (and the code agent) through a single gate that cannot be skipped.
- **D L0:** No approval mode exists to enable; the CLI has no approval flag. — [gui_agents/s3/cli_app.py:214](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L214); [gui_agents/s3/cli_app.py:312-317](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L312-L317) (verified)
  - *To reach the next level:* Ship approval on by default with an explicit, loudly named flag to disable it.
- **B L0:** Unapproved actions include irreversible GUI operations (send, delete, purchase) and arbitrary Python with sudo attempts; nothing offers undo or preview. — [gui_agents/s3/cli_app.py:214-215](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L214-L215); [gui_agents/s3/agents/grounding.py:434](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/grounding.py#L434) (verified)
  - *To reach the next level:* Add checkpoints or previews for external actions and bound quantities.
- **Cap:** C2-POWERBYPASS — The single most powerful path, exec of model-generated Python at cli_app.py:215, runs with no gate in the default CLI.

### C3 Tool & action scoping — 0.00 (high)

The agent advertises a small set of GUI actions (click, type, hotkey, open, scroll), but that set is not enforced. The model's reply is evaluated as Python, and the only check is a regular expression that the text contains exactly one call starting with 'agent.', which any payload can satisfy by nesting code inside the arguments. The action builders also paste arguments into code strings without escaping. In practice the tool surface is arbitrary Python on the user's machine.

- **S L0:** Raw passthrough: model text is eval'd; the only check is a regex that one 'agent.' call appears, and hotkey/set_cell_values splice unescaped arguments into executable code. — [gui_agents/s3/utils/common_utils.py:31](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/common_utils.py#L31); [gui_agents/s3/utils/common_utils.py:178](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/common_utils.py#L178); [gui_agents/s3/agents/grounding.py:628](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/grounding.py#L628) (verified)
  - *To reach the next level:* Parse the action with ast and dispatch only allowlisted agent methods with literal, type-checked arguments.
- **C L0:** No action validates its inputs in code. — searched `rg -n -i 'allowlist|whitelist|denylist|blacklist|realpath|validate'` in `gui_agents/s3` → 1 hits (Single hit is prompt text in procedural_memory.py:315 ('invalidated'); no argument validation exists.) (verified)
  - *To reach the next level:* Validate arguments for every action in a shared dispatcher.
- **D L0:** All GUI actions are enabled by default and eval gives unrestricted code execution regardless of the opt-in code-agent flag; only set_cell_values/call_code_agent are hidden from the prompt. — [gui_agents/s3/agents/worker.py:64-73](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/worker.py#L64-L73); [gui_agents/s3/utils/common_utils.py:31](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/common_utils.py#L31) (verified)
  - *To reach the next level:* Default to a narrow tool set and require explicit enabling for write or exec capabilities.
- **B L0:** A misused action is general-purpose code against the whole machine. — [gui_agents/s3/utils/common_utils.py:31](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/common_utils.py#L31) (verified)
  - *To reach the next level:* Scope actions to a workspace or dedicated VM with bounded operations.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

Model output is executed twice inside the agent's own Python process: eval() turns the model's text into an action string, then exec() runs that string. There is no sandbox, container, separate user or restricted interpreter on either step, and the optional code agent also runs bash and Python directly on the host. The README advises running untrusted tasks in a sandbox, but the code provides none, so any escape lands in a process holding the API keys and the user's full desktop.

- **S L0:** No isolation: in-process eval and exec of model-generated code on the host; the opt-in code agent uses same-user subprocesses. — [gui_agents/s3/utils/common_utils.py:31](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/common_utils.py#L31); [gui_agents/s3/cli_app.py:214-215](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L214-L215); searched `rg -n -i 'sandbox|docker|seccomp|firejail|bwrap|chroot'` in `gui_agents/s3` → 0 hits (No isolation primitive anywhere in the scored S3 package.) (verified)
  - *To reach the next level:* Execute actions inside a hardened container or VM boundary the model cannot redefine.
- **C L0:** No execution path is isolated: eval in format checking, eval in action creation, exec in the CLI, and opt-in bash/python subprocesses all run on the host. — [gui_agents/s3/utils/formatters.py:30-35](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/formatters.py#L30-L35); [gui_agents/s3/utils/local_env.py:15-19](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/local_env.py#L15-L19) (verified)
  - *To reach the next level:* Send every execution path through one sandbox that fails closed.
- **D L0:** No sandbox exists to enable; README only advises users to run in a sandbox themselves. — [gui_agents/s3/cli_app.py:214-215](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L214-L215); [README.md:244](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/README.md#L244) (verified)
  - *To reach the next level:* Ship a sandboxed default backend.
- **B L0:** Host-equivalent: code runs in the agent process with API keys in the environment, the user's home directory, and full network access. — [gui_agents/s3/utils/common_utils.py:31](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/common_utils.py#L31); [gui_agents/s3/core/engine.py:43](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/core/engine.py#L43) (verified)
  - *To reach the next level:* Run in an ephemeral environment with no secrets, workspace-only files and restricted egress.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

The agent reads whatever is on screen, including web pages, emails and documents written by others, and sends it straight to the model alongside the user's task. Nothing marks this content as untrusted or limits what the model may do after seeing it. Because the next action is arbitrary Python run without approval, a page that hijacks the model can make it exfiltrate files and keys over the network and take irreversible actions, all unattended.

- **S L0:** Nothing structural limits a hijacked agent; there is no detection, tagging, or capability reduction after reading untrusted content. — [gui_agents/s3/agents/worker.py:305-306](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/worker.py#L305-L306); searched `rg -n -i 'untrusted|inject|provenance'` in `gui_agents/s3` → 0 hits (No untrusted-content handling, tagging, or provenance.) (verified)
  - *To reach the next level:* Disable or gate egress and state-changing actions once untrusted content has been read.
- **C L0:** Screenshots and OCR text of any on-screen content enter context with the same standing as the user's instruction. — [gui_agents/s3/cli_app.py:165](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L165); [gui_agents/s3/agents/worker.py:305-306](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/worker.py#L305-L306) (verified)
  - *To reach the next level:* Distinguish untrusted sources (screen content, code-agent output) from the principal's instruction.
- **D L0:** No control exists, so nothing is on by default. — searched `rg -n -i 'untrusted|inject|provenance'` in `gui_agents/s3` → 0 hits (No untrusted-content handling, tagging, or provenance.) (verified)
  - *To reach the next level:* Ship an untrusted-content policy on by default.
- **B L0:** A hijacked model can leak secrets and act irreversibly with no human: its action is eval'd as arbitrary Python and exec'd without approval. — [gui_agents/s3/utils/common_utils.py:31](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/common_utils.py#L31); [gui_agents/s3/cli_app.py:214-215](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L214-L215) (verified)
  - *To reach the next level:* Require human approval for egress and irreversible actions in sessions that read untrusted content.
- **Cap:** C5-WORSTCASE — In the default configuration a hijacked agent can both exfiltrate and take irreversible actions unattended via eval/exec of its output.

### C6 Memory, context & configuration integrity — 1.00 (high)

The S3 agent keeps no memory between runs and auto-loads no workspace files. Its 'procedural memory' is a set of static prompts in the package, its text buffer lives only in process memory, and its log files are never read back. There is therefore no persistent path for poisoned content to shape future sessions through the agent itself, although arbitrary code execution (scored under code execution) can of course modify the user's files.

- **Structural absence:** searched `rg -n -i 'AGENTS\.md|CLAUDE\.md|cursorrules|vector|chroma|faiss|sqlite|embedding|save_to_disk|json\.dump'` in `gui_agents/s3` → 0 hits (No memory store, retrieval index, or auto-loaded instruction file in S3.); searched `rg -n 'load_dotenv|json\.load|pickle|\.read\('` in `gui_agents/s3` → 5 hits (Hits are image reads (mllm.py:119, comparative_judge.py:40), terminal stdin (cli_app.py:36), and json.loads inside the set_cell_values exec template (grounding.py:72-73); nothing is read back from persisted agent state.); searched `rg -n 'open\('` in `gui_agents/s3` → 7 hits (All seven are image reads/Image.open or the ACI method named open (grounding.py:392); S3 writes no files besides logs, which are never read back.)
- **Notes:** grounding_agent.notes (grounding.py:204,471) survives agent.reset() across queries in one interactive CLI process, but it is in-memory only. Legacy S1/S2 packages have a pickle-backed knowledge base (not scored). osworld_setup scripts call load_dotenv() but are benchmark harnesses, not the shipped CLI.

### C7 Third-party extensions — 0.00 (high)

Agent S has no plugin or MCP system, but its type action generates code that, when the pyperclip package is missing, installs it from PyPI at runtime (unpinned) and runs apt-get under sudo using a hardcoded benchmark password. Because pyperclip is not a declared dependency, this fires on a fresh install the first time the agent types, without asking, and the freshly installed package is imported into the agent's own process.

- **S L0:** Executes an unpinned PyPI install (and a sudo apt-get) automatically from generated action code. — [gui_agents/s3/agents/grounding.py:434-435](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/grounding.py#L434-L435); searched `rg -n 'pyperclip'` in `setup.py requirements.txt` → 0 hits (pyperclip is not a declared dependency, so the runtime install path in the type action fires on a fresh install.) (verified)
  - *To reach the next level:* Declare pyperclip as a pinned dependency and remove runtime installs.
- **C L0:** No verification of any runtime-installed code. — [gui_agents/s3/agents/grounding.py:434-435](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/grounding.py#L434-L435) (verified)
  - *To reach the next level:* Verify or forbid every runtime install path.
- **D L0:** The install happens automatically without consent. — [gui_agents/s3/agents/grounding.py:434-435](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/grounding.py#L434-L435); searched `rg -n 'importlib|entry_points|plugin|mcp'` in `gui_agents/s3` → 0 hits (No plugin or MCP loader; the only runtime third-party code path is the automatic pyperclip install.) (verified)
  - *To reach the next level:* Require explicit user consent showing the exact package before any install.
- **B L0:** The installed package is imported in-process by exec'd code, with the agent's credentials and full environment. — [gui_agents/s3/cli_app.py:214-215](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L214-L215); [gui_agents/s3/agents/grounding.py:434-435](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/grounding.py#L434-L435) (verified)
  - *To reach the next level:* Isolate third-party code in a separate process with a scrubbed environment.
- **Cap:** C7-RCELOAD — By default the type action's generated code pip-installs an unpinned PyPI package and imports it in-process without consent.

### C8 Secrets & sensitive-data protection — 0.00 (high)

API keys are read from environment variables or passed as command-line arguments, where they show up in process listings, and nothing anywhere masks or redacts secrets. The CLI turns on debug-level logging to a logs folder in the current directory and records every model plan plus the code agent's full code and output. Full-screen screenshots, which can show passwords or private messages, go to the model provider unfiltered. Secret handling in a bundled integration wrapper is not locked down.

- **S L0:** No masking or redaction anywhere; keys accepted on argv; a committed credential (sudo password) is placed in model prompts. — [gui_agents/s3/cli_app.py:248-253](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L248-L253); [gui_agents/s3/memory/procedural_memory.py:110](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/memory/procedural_memory.py#L110); searched `rg -n -i 'redact|mask|SecretStr|sanitiz'` in `gui_agents/s3` → 0 hits (No redaction or secret masking anywhere.) (verified)
  - *To reach the next level:* Add redaction for logs and model-bound content and stop taking keys on the command line.
- **C L0:** No path is protected: logs, model-bound screenshots and subprocess environments all carry raw content. — [gui_agents/s3/utils/local_env.py:15-19](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/local_env.py#L15-L19) (verified)
  - *To reach the next level:* Protect at least logs and transcripts with redaction.
- **D L0:** Verbose payload logging is on by default: root logger at DEBUG with a debug file handler, and plans and code-agent outputs logged at INFO into ./logs. — [gui_agents/s3/cli_app.py:91](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L91); [gui_agents/s3/cli_app.py:98-103](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L98-L103); [gui_agents/s3/agents/code_agent.py:35](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/code_agent.py#L35) (verified)
  - *To reach the next level:* Default to content-free INFO logging and make payload logging opt-in.
- **B L0:** Long-lived provider API keys sit in the environment of the process that evals model output, reachable by the model. — [gui_agents/s3/core/engine.py:43](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/core/engine.py#L43); [gui_agents/s3/utils/common_utils.py:31](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/common_utils.py#L31) (verified)
  - *To reach the next level:* Keep keys out of the process that executes model code, or use short-lived scoped keys.
- **Cap:** none
- **Notes:** C8-MODELSECRETS not applied: the sudo password in the prompt is a public benchmark credential, not the operator's secret. No telemetry SDK was found.

### C9 Audit & traceability — 0.33 (high)

The CLI writes timestamped text logs that include each model plan (which contains the chosen action call) and, when enabled, the code agent's commands and outputs. However, the actual code that gets executed is only printed to the terminal, not logged, results are not recorded per action, and screenshots are not saved, so a run cannot be fully reconstructed. Logs go to a folder in the current directory where the agent's own code can modify or delete them.

- **S L1:** Unstructured, timestamped text logs of plans and code-agent steps; the executed exec_code and action results are not recorded. — [gui_agents/s3/agents/worker.py:322](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/worker.py#L322); [gui_agents/s3/cli_app.py:208](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L208); [gui_agents/s3/agents/code_agent.py:35](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/code_agent.py#L35) (verified)
  - *To reach the next level:* Write a structured per-action record with arguments, executed code, result status and timestamps.
- **C L2:** Every GUI action passes through the logged plan and the code agent logs its steps; there is no approval record (no approvals exist) or cross-agent correlation. — [gui_agents/s3/agents/worker.py:322](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/worker.py#L322); [gui_agents/s3/agents/code_agent.py:35](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/code_agent.py#L35) (verified)
  - *To reach the next level:* Record approvals/denials and correlate sub-agent (code agent) records with the parent step.
- **D L1:** On by default but written to ./logs in the working directory, editable by the model's own eval'd code. — [gui_agents/s3/cli_app.py:95](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L95); [gui_agents/s3/cli_app.py:98-103](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L98-L103) (verified)
  - *To reach the next level:* Write logs outside the agent's reach via a component the model can't control.
- **B L1:** Best-effort logging: actions proceed regardless of logging failures, and without executed code or screenshots the trajectory can't be replayed. — [gui_agents/s3/cli_app.py:98-103](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L98-L103); [gui_agents/s3/cli_app.py:208](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L208) (verified)
  - *To reach the next level:* Flush a complete record per action, including executed code, before continuing.
- **Cap:** none

### C10 Limits & kill switch — 0.25 (high)

The CLI stops after 15 steps per task, a hardcoded loop bound, but has no time or spend limit. The step bound is in the CLI loop only; the AgentS3 SDK class has none. Model-chosen actions run without timeouts, the wait action sleeps for whatever duration the model picks, and the opt-in code agent gets a fresh 20-step budget on every call, with no timeout on its Python runs. Ctrl+C pauses and a second Ctrl+C exits the CLI, but processes started by evaluated code can outlive it.

- **S L1:** Iteration cap only (15 steps); no wall-clock, token or cost cap on the default path. — [gui_agents/s3/cli_app.py:160](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L160); searched `rg -n -i 'cost|price|wall|deadline|max_time|timeout'` in `gui_agents/s3` → 17 hits (Hits are per-call backoff max_time=60 on LLM retries (engine.py, 9 hits), an unused Azure cost counter (engine.py:272,310), cost_this_turn=0 never compared (worker.py:87), the opt-in bash timeout (local_env.py 4 hits, code_agent.py:39), and prompt text; no session wall-clock or spend cap.) (verified)
  - *To reach the next level:* Add a session wall-clock limit and a token/cost budget enforced in code.
- **C L1:** The cap applies to the top-level loop only; exec'd actions, opt-in python runs and model-chosen waits have no timeout. — [gui_agents/s3/cli_app.py:214-215](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L214-L215); [gui_agents/s3/utils/local_env.py:50-53](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/local_env.py#L50-L53); [gui_agents/s3/agents/grounding.py:655](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/grounding.py#L655) (verified)
  - *To reach the next level:* Apply timeouts to every action and count code-agent steps against the session budget.
- **D L1:** The 15-step bound lives only in the CLI loop (the AgentS3 SDK has no step limit), each call_code_agent starts a fresh 20-step budget, and eval'd code can start threads or processes that keep acting outside the cap. — [gui_agents/s3/cli_app.py:160](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L160); [gui_agents/s3/agents/code_agent.py:138](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/code_agent.py#L138); [gui_agents/s3/agents/code_agent.py:127](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/agents/code_agent.py#L127) (verified)
  - *To reach the next level:* Enforce a step/time budget inside the agent itself that delegation and in-process code can't reset or escape.
- **B L1:** No time or spend ceiling; Ctrl+C exits the loop via sys.exit, but background processes spawned by eval'd code keep running. — [gui_agents/s3/cli_app.py:68-70](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/cli_app.py#L68-L70); [gui_agents/s3/utils/common_utils.py:31](https://github.com/simular-ai/Agent-S/blob/3aa272d23d2994c7bbde1acbbe0ef8e8d06b8693/gui_agents/s3/utils/common_utils.py#L31) (verified)
  - *To reach the next level:* Add tight time/cost ceilings and kill the process group on stop.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Full-screen screenshots of any web page, email or document (gui_agents/s3/cli_app.py:165) · [B] sensitive data/systems: User's files, logged-in sessions and API keys in os.environ reachable from in-process eval (gui_agents/s3/utils/common_utils.py:31, gui_agents/s3/core/engine.py:43) · [C] state change / egress: Unapproved exec of model-generated Python with full network and GUI control (gui_agents/s3/cli_app.py:215) · Same default session? Yes

## Highest-impact improvements
1. Call the existing show_permission_dialog (with the exact exec code) before every exec in cli_app.py, on by default with an explicit flag to disable. — C2 S L0→L3, +0.225 before caps (Playbook 5)
2. Replace eval() of model text with ast parsing that dispatches only allowlisted agent.<action> methods with literal, type-checked arguments, and escape all arguments spliced into generated code. — C3 S L0→L3, +0.225 before caps (Playbook 3)
3. Declare pyperclip as a pinned dependency and remove the runtime sudo apt-get / pip install from the type action and the sudo password from prompts. — C7 S L0→L2, +0.150 before caps (Playbook 3)
4. Default logging to INFO without payloads and stop accepting API keys on argv. — C8 D L0→L1, +0.050 before caps (Playbook 4)
5. Add a session wall-clock limit, per-action timeouts and a shared budget across code-agent calls. — C10 S L1→L2, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scope is the Agent S3 package (gui_agents/s3) and the agent_s entry point; legacy S1/S2/S2.5 packages, osworld_setup benchmark harnesses and the BBoN judge scripts were only skimmed. Legacy S1/S2 use pickle-backed knowledge stores and share the same unused permission dialog.
- The integrations/openclaw wrapper was reviewed only for secret handling.
- Behaviour of third-party SDKs under the root DEBUG logger (e.g., request-payload debug logs from openai/anthropic/httpx) was inferred, not verified.
- Platform-specific behaviour (pyautogui, osascript, zenity, wmctrl) was not examined beyond the code paths cited.
- No reviewer-directed prompt-injection text was found in README, models.md or integration docs.
