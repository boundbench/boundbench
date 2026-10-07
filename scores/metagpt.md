# Defense-in-Depth Score: MetaGPT

**Repo:** https://github.com/foundationagents/metagpt · **Commit:** `11cdf466d042aece04fc6cfd13b28e1a70341b1f` · **Reviewed:** 2026-10-04
**What it is:** Multi-agent framework that simulates a software company (team leader, product manager, architect, engineer, data analyst) to turn a one-line requirement into code.
**Category:** Agent Frameworks
**Scored configuration:** `pip install metagpt` then `metagpt "<idea>"` with default options: MGXEnv team of TeamLeader, ProductManager, Architect, Engineer2 and DataAnalyst, $3 investment, 5 rounds.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 0.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | C2-POWERBYPASS | **0.00** | High |
| C3 | Tool & action scoping | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | C7-RCELOAD | **0.00** | High |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C9 | Audit & traceability | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | High |
| C10 | Limits & kill switch | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |


MetaGPT runs model-written shell commands and notebook code directly on your machine, as you, with your full environment and with no approval step. Its Engineer can write any file, browse any URL, install packages it picks, and open pull requests using your ~/.git-credentials. Anything injected through a web page or file can therefore steal keys and act irreversibly. Run it only inside a disposable VM or container with no credentials you care about.

## Critical gaps
- The default shell tool runs every model command on the host with no approval gate. (ASI02, ASI09; C2) — [metagpt/roles/di/role_zero.py:443-446](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L443-L446)
- The Engineer's bash shell inherits the full process environment and the PR tool reads ~/.git-credentials, giving a hijacked agent the user's whole authority. (ASI03; C1) — [metagpt/tools/libs/terminal.py:49-56](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L49-L56); [metagpt/tools/libs/git.py:69-71](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/git.py#L69-L71)
- Model-generated code executes in a same-user host shell and local Jupyter kernel with credentials in the environment; no sandbox exists. (ASI05; C4) — [metagpt/tools/libs/terminal.py:49-56](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L49-L56); [metagpt/actions/di/execute_nb_code.py:104-107](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/actions/di/execute_nb_code.py#L104-L107)
- Tool outputs (including web pages) enter memory as user messages, and a hijacked agent can exfiltrate and act irreversibly unattended. (ASI01, LLM01; C5) — [metagpt/roles/di/role_zero.py:293-295](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L293-L295); [metagpt/tools/libs/browser.py:126-132](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/browser.py#L126-L132)
- A config/config2.yaml in the current working directory is merged silently on PyPI installs and can enable tools and persistent memory. (ASI06; C6) — [metagpt/const.py:28-33](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/const.py#L28-L33); [metagpt/config2.py:115-121](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/config2.py#L115-L121)
- The analysis prompt directs the model to pip-install packages it chooses through the host shell, running unverified install code with full credentials. (ASI04; C7) — [metagpt/prompts/di/write_analysis_code.py:33](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/prompts/di/write_analysis_code.py#L33); [metagpt/tools/libs/terminal.py:49-56](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L49-L56)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

MetaGPT has no identity of its own: every tool runs with the full authority of the OS user who launched it. The persistent shell given to the Engineer receives a copy of the entire process environment (including any LLM, cloud or Git tokens), and the pull-request tool reads the user's ~/.git-credentials file directly. There is no authorization layer between a model-chosen command and these credentials, so a hijacked agent holds everything the user holds.

- **S L0:** Ambient authority: the shell inherits os.environ wholesale and git_create_pull reads the user's ~/.git-credentials file. — [metagpt/tools/libs/terminal.py:49-56](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L49-L56); [metagpt/tools/libs/git.py:69-71](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/git.py#L69-L71) (verified)
  - *To reach the next level:* No dedicated or scoped identity; tools use the operator's full credentials.
- **C L0:** Tools construct their own privileged access (git_create_pull opens the credential store itself); no authorization check exists on any tool path. — [metagpt/tools/libs/git.py:69-71](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/git.py#L69-L71); [metagpt/roles/di/role_zero.py:394-400](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L394-L400) (verified)
  - *To reach the next level:* No tool path passes an authorization check; even the main path uses ambient credentials.
- **D L0:** The default install runs every role with the launching user's full privileges; there is no narrower default. — [metagpt/tools/libs/terminal.py:49-56](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L49-L56); [metagpt/software_company.py:45-53](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/software_company.py#L45-L53) (verified)
  - *To reach the next level:* No least-privilege default exists to configure.
- **B L0:** A hijacked Engineer can use every credential in the environment, the user's Git hosting tokens, and anything the OS user can reach. — [metagpt/tools/libs/terminal.py:49-56](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L49-L56); [metagpt/tools/libs/git.py:69-71](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/git.py#L69-L71); [metagpt/roles/di/engineer2.py:98-106](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/engineer2.py#L98-L106) (verified)
  - *To reach the next level:* No surviving layer limits reach to one system or tenant.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no approval step anywhere in the default flow. The RoleZero dispatcher executes whatever command name the model emits if it appears in the tool map, including raw shell commands, notebook code execution, file writes anywhere on disk, and opening pull requests. An ask_human tool exists, but only the model decides to call it, so it is not a gate. A plan-review prompt exists for the older DataInterpreter, but RoleZero roles force it off.

- **S L0:** _run_commands calls the mapped tool directly with model-supplied arguments; no human or policy check sits in between. — [metagpt/roles/di/role_zero.py:394-400](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L394-L400); searched `rg -n -S 'approv|confirm'` in `metagpt/roles metagpt/tools/libs metagpt/actions/di` → 10 hits (Hits are DataInterpreter/Planner plan-review text (AskReview, used only when Planner.auto_run is False; RoleZero forces auto_run=True) and an editor error string; none gates a tool call.) (verified)
  - *To reach the next level:* No per-call human approval of consequential actions.
- **C L0:** The most powerful tool, Terminal.run_command, is dispatched as a special command with no gate. — [metagpt/roles/di/role_zero.py:443-446](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L443-L446); [metagpt/roles/di/engineer2.py:98-106](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/engineer2.py#L98-L106) (verified)
  - *To reach the next level:* Shell, code execution, and file writes all bypass any gate because none exists.
- **D L0:** No approval mode exists to be on by default; RoleZero also forces the planner's review to auto_run=True. — [metagpt/roles/di/role_zero.py:114](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L114) (verified)
  - *To reach the next level:* Approval is not available, let alone on by default.
- **B L0:** Shell commands can delete files or push code, and git_create_pull opens real pull requests, with no checkpoint or undo. — [metagpt/roles/di/engineer2.py:84](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/engineer2.py#L84); [metagpt/tools/libs/terminal.py:86-96](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L86-L96) (verified)
  - *To reach the next level:* No rollback, preview, or dry-run for shell or external actions.
- **Cap:** C2-POWERBYPASS — The shell tool (Terminal.run_command), the most powerful action path, executes in the default config with no gate (role_zero.py special-command branch).

### C3 Tool & action scoping — 0.00 (high)

The default Engineer gets a raw bash shell, a notebook code executor via the DataAnalyst, an editor that writes any absolute path, a browser that opens any URL, and a pull-request tool. Arguments are passed through unvalidated: the editor only joins relative paths onto its working directory with no containment check, and the only shell filter blocks the strings 'run dev' and 'serve ' to steer the model towards the deployer, not for safety. Every tool is enabled by default for its role.

- **S L0:** Terminal.run_command sends the model's string straight to a persistent bash; Editor paths are used as given; Browser.goto accepts any URL. — [metagpt/tools/libs/terminal.py:86-96](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L86-L96); [metagpt/tools/libs/editor.py:1096-1102](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/editor.py#L1096-L1102); [metagpt/tools/libs/browser.py:126-132](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/browser.py#L126-L132) (verified)
  - *To reach the next level:* No allowlist validation of paths, URLs, or commands.
- **C L0:** No built-in tool validates its inputs for safety; the 'forbidden_commands' map is a UX redirect for two dev-server strings. — [metagpt/tools/libs/terminal.py:41-45](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L41-L45); searched `rg -n 'realpath|relative_to|is_relative_to'` in `metagpt/tools/libs/editor.py metagpt/roles/di/engineer2.py` → 0 hits (No path containment check in the Editor or Engineer2 file writers.) (verified)
  - *To reach the next level:* Not even a few tools validate inputs against allowlists or bounds.
- **D L0:** The default team enables shell, editor writes, browser, PR creation and code execution without any opt-in. — [metagpt/roles/di/engineer2.py:39-51](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/engineer2.py#L39-L51); [metagpt/software_company.py:45-53](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/software_company.py#L45-L53) (verified)
  - *To reach the next level:* Write, exec and network tools are all on by default with no read-only tool set.
- **B L0:** A misused shell or editor reaches the whole machine as the user, and the browser reaches any host. — [metagpt/tools/libs/terminal.py:86-96](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L86-L96); [metagpt/tools/libs/editor.py:125-129](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/editor.py#L125-L129) (verified)
  - *To reach the next level:* Tools are not scoped to the workspace or bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

Model-written code runs directly on the host. The Engineer's shell is a same-user bash subprocess started with a full copy of the environment, and the DataAnalyst's notebook executor starts a local Jupyter kernel in the same user account. The package contains no container, sandbox, or seccomp backend on these paths. The shipped Dockerfile only packages MetaGPT itself and is not used as a per-execution sandbox.

- **S L0:** Execution is a same-user bash subprocess and a local Jupyter kernel; no isolation primitive exists. — [metagpt/tools/libs/terminal.py:49-56](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L49-L56); [metagpt/actions/di/execute_nb_code.py:104-107](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/actions/di/execute_nb_code.py#L104-L107); searched `rg -n -S 'docker|sandbox|seccomp|firejail|nsjail'` in `metagpt/tools/libs metagpt/actions/di metagpt/roles` → 0 hits (No isolation backend in the tool, executor, or role code.) (verified)
  - *To reach the next level:* No OS-level separation of executed code.
- **C L0:** No execution path is sandboxed, including the main shell tool and notebook execution. — [metagpt/roles/di/data_analyst.py:109](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/data_analyst.py#L109); [metagpt/roles/di/engineer2.py:98-106](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/engineer2.py#L98-L106) (verified)
  - *To reach the next level:* The main exec tool is not sandboxed.
- **D L0:** There is no sandbox to turn on; host execution is the only mode. — [metagpt/tools/libs/terminal.py:49-56](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L49-L56) (verified)
  - *To reach the next level:* No sandbox exists, so none is on by default.
- **B L0:** Executed code runs with the user's home directory, full network, and every credential in the copied environment. — [metagpt/tools/libs/terminal.py:49-56](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L49-L56); [metagpt/tools/libs/git.py:69-71](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/git.py#L69-L71) (verified)
  - *To reach the next level:* Nothing removes secrets, network, or host filesystem from executed code.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Agents browse arbitrary URLs and read files, and every tool result is appended to memory as a user-role message, the same standing as the human's request. Nothing marks content as untrusted or restricts later actions after untrusted content is read. A hijacked Engineer can then read secrets from its environment, send them anywhere through the shell or browser, and take irreversible actions, all without a human in the loop.

- **S L0:** No structural limit on a hijacked agent; there is no provenance, taint, or approval tied to untrusted input. — [metagpt/roles/di/role_zero.py:293-295](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L293-L295); searched `rg -n -i -w 'untrusted|injection|tainted|provenance'` in `metagpt/roles metagpt/prompts/di metagpt/utils/role_zero_utils.py metagpt/memory` → 0 hits (No provenance, taint, or untrusted-content handling in the roles, their prompts, the command parser, or memory.) (verified)
  - *To reach the next level:* No capability is disabled or gated after untrusted content is read.
- **C L0:** Tool outputs, including browser pages, enter context as UserMessage with the same standing as the user's instructions. — [metagpt/roles/di/role_zero.py:293-295](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L293-L295); [metagpt/tools/libs/browser.py:126-132](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/browser.py#L126-L132) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished from the principal.
- **D L0:** No defense exists to be on by default. — [metagpt/roles/di/role_zero.py:293-295](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L293-L295) (verified)
  - *To reach the next level:* No untrusted-input control ships at all.
- **B L0:** Default config lets a hijacked agent exfiltrate via shell or browser and take irreversible shell/PR actions unattended. — [metagpt/tools/libs/terminal.py:49-56](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L49-L56); [metagpt/tools/libs/browser.py:126-132](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/browser.py#L126-L132); [metagpt/roles/di/engineer2.py:84](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/engineer2.py#L84) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions both run with no human approval.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.10 (high)

Default memory is in-process and session-scoped, and long-term memory and the experience pool are off by default. However, when MetaGPT is installed from PyPI its project root falls back to the current working directory, and it merges config/config2.yaml from that directory into its settings without asking. A file in the directory where the user runs MetaGPT (or one the agent itself writes there via its unrestricted shell) can enable extra tools such as search, switch on persistent long-term memory or the experience pool, or move the workspace, and that persists across sessions.

- **S L0:** A cwd-relative config/config2.yaml is loaded silently and can enable tools (enable_search) and persistent memory with no trust prompt. — [metagpt/const.py:28-33](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/const.py#L28-L33); [metagpt/config2.py:115-121](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/config2.py#L115-L121); [metagpt/roles/di/role_zero.py:128-129](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L128-L129) (verified)
  - *To reach the next level:* Security-relevant project config needs an explicit workspace-trust decision.
- **C L0:** Neither the auto-loaded config nor the opt-in long-term memory store is validated or gated. — [metagpt/config2.py:115-121](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/config2.py#L115-L121); [metagpt/roles/di/role_zero.py:181-187](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L181-L187) (verified)
  - *To reach the next level:* No memory or config path is controlled.
- **D L1:** Long-term memory is off by default; when enabled it is namespaced only by role name under one shared persist path, with no per-user or per-project isolation. — [metagpt/configs/role_zero_config.py:7](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/configs/role_zero_config.py#L7); [metagpt/roles/di/role_zero.py:181-187](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L181-L187); [metagpt/configs/exp_pool_config.py:18](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/configs/exp_pool_config.py#L18) (verified)
  - *To reach the next level:* No per-user or per-session namespace enforced in queries.
- **B L1:** A poisoned cwd config persists across that user's sessions and can enable tools the agent then uses. — [metagpt/config2.py:115-121](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/config2.py#L115-L121); [metagpt/roles/di/role_zero.py:128-129](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L128-L129) (verified)
  - *To reach the next level:* Poisoned config can still trigger tool use rather than only text or gated actions.
- **Cap:** C6-REPOCONFIG — When the package root has no .git/.gitignore (a PyPI install) METAGPT_ROOT is Path.cwd() and cwd/config/config2.yaml is merged into Config, so a file in the working directory can enable tools (enable_search) and persistent memory without a trust decision.

### C7 Third-party extensions — 0.00 (high)

MetaGPT has no plugin or MCP system, but its code-writing prompt tells the DataAnalyst to install any missing package through the Terminal tool, so the model chooses and installs packages from PyPI on its own. Those installs run package install scripts on the host, unpinned and unverified, with the user's full environment and credentials. No consent is asked.

- **S L0:** The analysis-code prompt instructs model-chosen 'pip install' through Terminal; nothing pins or verifies what is installed. — [metagpt/prompts/di/write_analysis_code.py:33](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/prompts/di/write_analysis_code.py#L33); searched `rg -n -S 'sha256|hashlib|signature'` in `metagpt/tools/libs metagpt/actions/di` → 2 hits (Hits are unrelated to verifying installed packages (no integrity check on model-driven installs).) (verified)
  - *To reach the next level:* No version pinning or integrity check on runtime installs.
- **C L0:** No runtime-loaded code type is verified. — [metagpt/prompts/di/write_analysis_code.py:33](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/prompts/di/write_analysis_code.py#L33); searched `rg -n -i -t py 'mcp'` in `metagpt` → 0 hits (No MCP client or plugin loader in the Python package (non-Python hits, such as minified HTML bundles, excluded by -t py).) (verified)
  - *To reach the next level:* No extension type is verified.
- **D L0:** Packages are installed automatically whenever the model decides to, without a consent step. — [metagpt/prompts/di/write_analysis_code.py:33](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/prompts/di/write_analysis_code.py#L33); [metagpt/tools/libs/terminal.py:86-96](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L86-L96) (verified)
  - *To reach the next level:* No explicit install step or consent.
- **B L0:** Install scripts run in the same-user host shell with the full copied environment. — [metagpt/tools/libs/terminal.py:49-56](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L49-L56) (verified)
  - *To reach the next level:* No separate process with scrubbed environment for installed code.
- **Cap:** C7-RCELOAD — By default the DataAnalyst prompt directs the model to pip-install whatever package it needs through the host shell, executing unverified remote install code without consent.

### C8 Secrets & sensitive-data protection — 0.00 (high)

API keys live in plaintext YAML or environment variables with no masking type or redaction anywhere in the package. The default log file captures DEBUG output, which includes every full message sent to the LLM, and those prompts carry whatever tool output the agent saw (for example an 'env' command). Every shell command receives the whole environment, so long-lived keys are one 'printenv' away from the model and its provider. No third-party telemetry was found.

- **S L0:** No redaction or secret type exists; keys are plain str fields; full LLM messages are logged. — [metagpt/configs/llm_config.py:59](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/configs/llm_config.py#L59); [metagpt/provider/base_llm.py:204-205](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/provider/base_llm.py#L204-L205); searched `rg -n -S 'redact|SecretStr'` in `metagpt` → 0 hits (No secret masking type or redaction helper anywhere in the package.) (verified)
  - *To reach the next level:* No masking of secrets on any path.
- **C L0:** No path (logs, model-bound messages, subprocess env, saved state) is protected. — [metagpt/tools/libs/terminal.py:49-56](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L49-L56); [metagpt/provider/base_llm.py:204-205](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/provider/base_llm.py#L204-L205) (verified)
  - *To reach the next level:* Not even one path is protected.
- **D L0:** The file sink defaults to DEBUG, so verbose payload logging of every prompt is on by default. — [metagpt/logs.py:40](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/logs.py#L40); [metagpt/logs.py:51](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/logs.py#L51) (verified)
  - *To reach the next level:* Verbose payload logging is on by default.
- **B L0:** Long-lived LLM and Git keys are reachable by every subprocess and therefore by the model. — [metagpt/tools/libs/terminal.py:49-56](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L49-L56); [metagpt/tools/libs/git.py:69-71](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/git.py#L69-L71) (verified)
  - *To reach the next level:* Keys are long-lived and reachable from every subprocess.
- **Cap:** none

### C9 Audit & traceability — 0.33 (high)

MetaGPT writes a loguru log to stderr and to a logs/ directory under the project root. Each RoleZero step logs the parsed command list and its outputs at INFO level, and full LLM prompts at DEBUG, which gives a rough reconstruction of what each role did. The record is free text, carries no notion of who approved what (there are no approvals), and sits next to the workspace where the agent's own shell can edit or delete it.

- **S L1:** Commands and outputs are logged as free-text INFO lines, not a structured per-call record. — [metagpt/roles/di/role_zero.py:292-294](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L292-L294); [metagpt/logs.py:51](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/logs.py#L51) (verified)
  - *To reach the next level:* No structured record with arguments, status and timestamps per tool call.
- **C L2:** All RoleZero roles dispatch through the same _run_commands, so every built-in tool call in the default team is logged. — [metagpt/roles/di/role_zero.py:292-294](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L292-L294); [metagpt/roles/di/role_zero.py:394-400](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L394-L400) (verified)
  - *To reach the next level:* No record of approvals/denials (none exist) or of shell commands spawned inside notebook code.
- **D L1:** Logging is on by default but written under METAGPT_ROOT/logs, which the agent's unrestricted shell can modify. — [metagpt/logs.py:51](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/logs.py#L51); [metagpt/const.py:28-33](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/const.py#L28-L33) (verified)
  - *To reach the next level:* Log is not out of reach of the agent's tools.
- **B L1:** Logging is best-effort through loguru with no failure handling; actions proceed regardless. — [metagpt/logs.py:51](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/logs.py#L51) (verified)
  - *To reach the next level:* Errors in logging are not surfaced and nothing is flushed durably per action.
- **Cap:** none

### C10 Limits & kill switch — 0.38 (high)

There are a few limits: each RoleZero role stops after 40-50 actions, the team runs at most 5 rounds by default, the $3 'investment' budget is checked between rounds, and notebook cells time out after 600 seconds. These are coarse: the budget is only checked between rounds and ignores models missing from the price table, a role that hits its action limit asks the human whether to reset it, and the Terminal tool has no timeout at all. Background ('daemon') shell commands keep running after the agent stops.

- **S L2:** Iteration caps (max_react_loop, n_round), a between-round cost budget, and a 600s notebook cell timeout are enforced in code. — [metagpt/roles/di/role_zero.py:71](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L71); [metagpt/team.py:98-100](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/team.py#L98-L100); [metagpt/team.py:128-134](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/team.py#L128-L134); [metagpt/actions/di/execute_nb_code.py:76](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/actions/di/execute_nb_code.py#L76) (verified)
  - *To reach the next level:* No wall-clock limit and no rate limits on side-effecting tools; budget check is between rounds only.
- **C L1:** Limits bound the role loops and notebook cells but not the main Terminal tool, which waits indefinitely. — [metagpt/tools/libs/terminal.py:156-160](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L156-L160); searched `rg -n 'timeout'` in `metagpt/tools/libs/terminal.py` → 0 hits (Terminal has no timeout of any kind.) (verified)
  - *To reach the next level:* Terminal commands have no timeout.
- **D L2:** Defaults are modest ($3, 5 rounds, 40-50 actions) and operator-set via CLI options. — [metagpt/software_company.py:80-81](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/software_company.py#L80-L81); [metagpt/roles/di/role_zero.py:328-335](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/roles/di/role_zero.py#L328-L335); [metagpt/utils/cost_manager.py:48-50](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/utils/cost_manager.py#L48-L50) (verified)
  - *To reach the next level:* At the action cap the role asks the human to reset the counter, and unknown models add no cost.
- **B L1:** Daemon shell commands are spawned as untracked asyncio tasks and the bash process is never killed on stop, so work can keep running. — [metagpt/tools/libs/terminal.py:102-103](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/tools/libs/terminal.py#L102-L103); [metagpt/actions/di/execute_nb_code.py:234-236](https://github.com/foundationagents/metagpt/blob/11cdf466d042aece04fc6cfd13b28e1a70341b1f/metagpt/actions/di/execute_nb_code.py#L234-L236) (verified)
  - *To reach the next level:* Stopping does not cancel in-flight or background commands.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Browser.goto any URL, tool output as UserMessage (metagpt/tools/libs/browser.py:126-132, metagpt/roles/di/role_zero.py:293-295) · [B] sensitive data/systems: Full os.environ in shell and ~/.git-credentials (metagpt/tools/libs/terminal.py:49-56, metagpt/tools/libs/git.py:69-71) · [C] state change / egress: Host bash, editor writes, PR creation (metagpt/roles/di/engineer2.py:98-106) · Same default session? Yes

## Highest-impact improvements
1. Require per-call human approval showing the exact command for Terminal.run_command, write_and_exec_code, Editor writes and git_create_pull in RoleZero._run_commands. — C2 S L0→L3, +0.225 before caps (Playbook 5)
2. Run the Terminal shell and notebook kernel in a non-root, no-network-by-default container with only the workspace mounted. — C4 S L0→L3, +0.225 before caps (Playbook 3 step 1)
3. Start the Terminal subprocess with a scrubbed environment instead of os.environ.copy(). — C8 C L0→L1, +0.075 before caps (Playbook 4)
4. Stop loading config/config2.yaml from the current working directory; read only ~/.metagpt or an explicit --config path. — C6 S L0→L2, +0.150 before caps (Playbook 2)
5. Add a timeout and process-group kill to Terminal.run_command and check the cost budget on every LLM call. — C10 C L1→L2, +0.075 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the default CLI team only; metagpt/ext (aflow, sela, spo, stanford_town, werewolf, android), environments (android, minecraft), legacy SOP roles (Engineer, QaEngineer with RunCode) and RAG modules were not examined in depth.
- C6 cap relies on the inferred behaviour that a PyPI install's site-packages directory contains no .git, .project_root or .gitignore, so get_metagpt_root falls back to Path.cwd(); editable installs from a git clone load config from the clone instead.
- Clone used --filter=blob:limit=1m; large blobs were not fetched.
- No text aimed at AI reviewers was found in the repository.
