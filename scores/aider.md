# Defense-in-Depth Score: Aider

**Repo:** https://github.com/Aider-AI/aider · **Commit:** `5dc9490bb35f9729ef2c95d00a19ccd30c26339c` · **Reviewed:** 2026-10-03
**What it is:** AI pair programming in your terminal
**Category:** Coding
**Scored configuration:** Interactive `aider` CLI run inside a git repository, no flags, fresh pip install (code chat mode, auto-commits, auto-lint and shell-command suggestions on).
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions no · sub agents opt-in · external communication no

## Score: 3.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C2 | Approval gates | L2 | L2 | L1 | L2 | 0.45 | G2 | **0.25** | High |
| C3 | Tool & action scoping | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C4 | Code-execution isolation | L2 | L3 | L0 | L1 | 0.42 | G1 | **0.42** (alt) | High |
| C5 | Untrusted input blast radius | L2 | L1 | L1 | L0 | 0.28 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L0 | L1 | L1 | L1 | 0.17 | C6-REPOCONFIG | **0.17** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C9 | Audit & traceability | L1 | L2 | L1 | L2 | 0.38 | — | **0.38** | High |
| C10 | Limits & kill switch | L2 | L1 | L2 | L2 | 0.42 | — | **0.42** | High |

Controls where a risk surface exists: 2.57 / 9.0 (29%); 1 criterion scored SA (surface absent).

Aider asks for an explicit yes before running any shell command and auto-commits every edit so it can be undone, but it has no sandbox and passes your full environment to everything it runs. The dominant risk is the repository itself: aider silently loads .aider.conf.yml and .env from the project folder, and those files can configure commands that run with no prompt, auto-approve other prompts, or redirect your API key. Only run it in repositories you trust, or check those files first.

## Critical gaps
- Repository-controlled .aider.conf.yml/.env are auto-loaded and can configure commands (test-cmd+auto-test, lint-cmd, load) that run without any approval prompt, bypassing the shell gate. (ASI09, ASI02, ASI06; C2) — [aider/main.py:464-476](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L464-L476); [aider/main.py:361-381](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L361-L381); [aider/main.py:1123-1124](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L1123-L1124); [aider/coders/base_coder.py:1616-1617](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L1616-L1617)
- All commands, including the automatic lint and any configured test command, run on the host as the user with the full environment and network; there is no sandbox by default. (ASI05; C4) — [aider/run_cmd.py:62-72](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/run_cmd.py#L62-L72); [aider/run_cmd.py:116](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/run_cmd.py#L116); [aider/linter.py:137-159](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/linter.py#L137-L159)
- Worst case in the default configuration: an attacker-controlled repository gets unattended code execution with the user's credentials and network via auto-loaded config (test-cmd+auto-test, load); a further default path is not confined either. (ASI01, LLM01, ASI05; C5) — [aider/main.py:464-476](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L464-L476); [aider/main.py:1123-1124](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L1123-L1124)
- Repository-level .aider.conf.yml, .env (override=True) and model-settings files are auto-loaded with no trust prompt and can configure commands, auto-approval and the API endpoint. (ASI06, ASI04; C6) — [aider/main.py:464-476](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L464-L476); [aider/main.py:361-381](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L361-L381); [aider/args.py:35-42](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/args.py#L35-L42)

## Criterion details

### C1 Identity & least privilege — 0.05 (high)

Aider runs as the user who launched it and makes no attempt to narrow that authority. The LLM API keys it is given are written into its own process environment, and every command it runs (approved shell commands, lint and test commands) inherits that full environment, including any cloud or Git credentials the user has. There is no authorization layer in code; the only boundary is the human approval prompt for shell commands, which is scored under approval gates. The credentials aider itself holds are LLM provider keys, so a hijack mostly risks spend and whatever the user's shell can reach.

- **S L0:** Ambient OS-user authority; LLM keys are exported to os.environ and the shell runs as the user with no narrowing. — [aider/main.py:612-616](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L612-L616); [aider/run_cmd.py:62-72](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/run_cmd.py#L62-L72) (verified)
  - *To reach the next level:* No scoped identity or credential narrowing for spawned commands; subprocesses get the full user environment.
- **C L0:** No authorization check exists on any action path; every subprocess (shell, lint, test) inherits the full environment. — searched `rg -n 'env='` in `aider/run_cmd.py aider/linter.py aider/coders/base_coder.py aider/commands.py` → 1 hits (the single hit is cmd_git (commands.py:979) passing a copy of os.environ to git; no subprocess gets a scrubbed environment); [aider/run_cmd.py:116](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/run_cmd.py#L116) (verified)
  - *To reach the next level:* No authorization layer even on the main path; subprocess environments are never scrubbed.
- **D L0:** The default install runs with the user's full privileges; there is no narrower default mode. — [aider/run_cmd.py:62-72](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/run_cmd.py#L62-L72); [aider/main.py:612-616](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L612-L616) (verified)
  - *To reach the next level:* No reduced-privilege default (e.g. scrubbed env, read-only mode) exists.
- **B L1:** Aider's own credentials are long-lived LLM provider keys; approved commands additionally reach everything the user's account can, with the per-command approval prompt as the surviving layer. — [aider/main.py:612-616](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L612-L616); [aider/coders/base_coder.py:2456-2462](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L2456-L2462) (verified)
  - *To reach the next level:* Keys are long-lived and unscoped; anything the user's shell can reach is reachable by an approved command.
- **Cap:** none

### C2 Approval gates — 0.25 (high)

Shell commands proposed by the model are shown verbatim and need an explicit yes for each batch; even the --yes-always flag cannot approve them. File edits work differently: once a user adds a file to the chat, the model's edits to it are applied with no further prompt (then auto-committed to git so they can be undone), and new or out-of-chat files only show the path, not the change. The big gap is that configuration files in the repository being worked on (.aider.conf.yml, .env) are loaded automatically and can set a test command with auto-test, a lint command, a --load command file, or yes-always, which run commands or approve prompts with no human in the loop.

- **S L2:** Shell commands get per-call approval showing the exact commands with an explicit yes required; file writes are approved per file by path only (or not at all for files already in chat). — [aider/coders/base_coder.py:2456-2462](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L2456-L2462); [aider/io.py:866-867](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/io.py#L866-L867); [aider/coders/base_coder.py:2198-2200](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L2198-L2200); [aider/coders/base_coder.py:2206-2209](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L2206-L2209) (verified)
  - *To reach the next level:* Approval of file writes does not show the diff, and in-chat files are edited with no per-call approval.
- **C L2:** Shell suggestions, new files, out-of-chat edits, URL fetches and file additions are gated; in-chat edits and auto-lint run without approval; test/lint/load commands from config run unprompted. — [aider/coders/base_coder.py:2198-2200](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L2198-L2200); [aider/coders/base_coder.py:1599-1600](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L1599-L1600); [aider/coders/base_coder.py:1616-1617](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L1616-L1617); [aider/commands.py:993-1016](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/commands.py#L993-L1016) (verified)
  - *To reach the next level:* Configured test, lint and --load commands execute without crossing the gate, and in-chat edits are not a read-only auto-approval.
- **D L1:** Approval is on by default, but a repo-local .aider.conf.yml or .env can set yes-always, test-cmd+auto-test or load, which auto-approve prompts or run commands without one. — [aider/main.py:464-476](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L464-L476); [aider/main.py:361-381](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L361-L381); [aider/args.py:35-42](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/args.py#L35-L42); [aider/args.py:760-764](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/args.py#L760-L764); [aider/main.py:1123-1124](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L1123-L1124) (verified)
  - *To reach the next level:* Security settings must not be settable by files in the workspace without a trust decision.
- **B L2:** Edits are auto-committed to git (with a dirty-commit first) and /undo reverts aider commits; shell commands are irreversible. — [aider/args.py:440-444](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/args.py#L440-L444); [aider/commands.py:553-554](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/commands.py#L553-L554) (verified)
  - *To reach the next level:* No rollback or preview for shell side effects; no rate limits on consequential actions.
- **Cap:** G2 — A .aider.conf.yml or .env in the repository being edited can set test-cmd with auto-test, lint-cmd or load, which run arbitrary commands without approval, or yes-always, which auto-approves non-shell prompts.

### C3 Tool & action scoping — 0.30 (high)

Aider's actions are text edits to files and model-suggested shell commands. Edit targets are resolved against the repo root and files ignored by git are skipped, but there is no containment check that keeps paths inside the repo (paths outside only trigger a confirmation). Shell commands are passed to the user's shell unchanged with no validation. Read-only 'ask' mode exists, but the default mode includes both file writes and shell suggestions.

- **S L1:** Edits are filtered only by gitignore/.aiderignore and a confirmation; shell strings are raw passthrough to $SHELL. — [aider/coders/base_coder.py:2201-2204](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L2201-L2204); [aider/utils.py:96-102](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/utils.py#L96-L102); [aider/run_cmd.py:116](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/run_cmd.py#L116) (verified)
  - *To reach the next level:* No resolved-path containment for edit targets and no argument validation for commands.
- **C L1:** All edit formats pass through prepare_to_edit/allowed_to_edit; shell commands and configured lint/test commands pass through nothing. — [aider/coders/base_coder.py:2269-2287](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L2269-L2287); [aider/run_cmd.py:62-72](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/run_cmd.py#L62-L72) (verified)
  - *To reach the next level:* Shell and configured commands are not validated at all.
- **D L2:** Chat modes (code, ask, architect) are selectable and shell suggestions can be disabled, but the default code mode includes write and exec. — [aider/args.py:807-811](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/args.py#L807-L811); [aider/args.py:163-166](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/args.py#L163-L166) (verified)
  - *To reach the next level:* Default mode is not read-only; write and shell suggestions are on by default.
- **B L1:** A misused shell command reaches the whole machine as the user; edits are mostly within the repo but paths outside can be approved. — [aider/run_cmd.py:116](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/run_cmd.py#L116); [aider/coders/base_coder.py:2226-2229](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L2226-L2229) (verified)
  - *To reach the next level:* No workspace scoping or quantity bounds on commands.
- **Cap:** none

### C4 Code-execution isolation — 0.42 (high)

Every command aider runs (approved shell commands, the automatic flake8 lint, configured test and lint commands) runs directly on the host as the user, with no sandbox and with the full environment. The project ships a Docker image that runs aider as a non-root user, which contains these commands to the mounted project folder, but it is an opt-in installation method with full network access and the API key passed in. One default execution path is not confined.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Same-user subprocess with shell=True or pexpect $SHELL -i -c; no isolation primitive anywhere. — [aider/run_cmd.py:62-72](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/run_cmd.py#L62-L72); [aider/run_cmd.py:116](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/run_cmd.py#L116); searched `rg -n -i 'sandbox|seccomp|landlock|firejail|bwrap|nsjail' --glob '*.py'` in `aider` → 0 hits (no isolation primitive anywhere in the Python package) (verified)
    - *To reach the next level:* No OS-level separation (container, low-priv user, sandbox profile) for executed commands.
  - **C L0:** No execution path is isolated: shell suggestions, auto-lint flake8, test-cmd and lint-cmd all run on the host. — [aider/linter.py:137-159](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/linter.py#L137-L159); [aider/commands.py:993-1016](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/commands.py#L993-L1016) (verified)
    - *To reach the next level:* Even the main shell path is not sandboxed.
  - **D L0:** There is no default sandbox; the only isolation is the opt-in Docker image. — searched `rg -n -i 'sandbox|seccomp|landlock|firejail|bwrap|nsjail' --glob '*.py'` in `aider` → 0 hits (no isolation primitive anywhere in the Python package) (verified)
    - *To reach the next level:* Isolation is not on by default.
  - **B L0:** Commands run with the user's home directory, ~/.ssh, cloud credentials, LLM keys in env, and unrestricted network. — [aider/run_cmd.py:62-72](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/run_cmd.py#L62-L72); [aider/main.py:612-616](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L612-L616) (verified)
    - *To reach the next level:* Nothing limits reach: no workspace-only filesystem, no network restriction, no secret scrubbing.
- **opt-in Docker image (paulgauthier/aider)** (alt; raw 0.42, cap G1 → 0.42) ← counted
  - **S L2:** Stock container running aider as a non-root appuser; no dropped capabilities, seccomp or read-only root defined. — [docker/Dockerfile:9](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/docker/Dockerfile#L9); [docker/Dockerfile:54](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/docker/Dockerfile#L54) (verified)
    - *To reach the next level:* Container is not hardened (no cap drop, no-new-privileges, read-only root, network policy).
  - **C L3:** Aider itself is the container entrypoint, so every command it spawns runs inside the container. — [docker/Dockerfile:56](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/docker/Dockerfile#L56); [docker/Dockerfile:54](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/docker/Dockerfile#L54) (verified)
    - *To reach the next level:* No fail-closed guarantee beyond the image itself; escape hatches are whatever the operator mounts.
  - **D L0:** Using the container is an installation choice; the default pip install has no isolation. — [docker/Dockerfile:56](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/docker/Dockerfile#L56) (verified)
    - *To reach the next level:* Not the default deployment.
  - **B L1:** Docs mount the project folder read-write and pass the API key; network is unrestricted. — [aider/website/docs/install/docker.md:24](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/website/docs/install/docker.md#L24) (verified)
    - *To reach the next level:* Network egress is unrestricted with credentials in the container.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.25 (high)

Aider reads untrusted text from the repository (files, repo map), web pages the user adds, and command and lint output, and inserts it into the conversation as user-role messages, with file contents explicitly labelled as trusted. Nothing distinguishes these sources. A hijacked model cannot fetch URLs or run shell commands without the user's explicit yes, but it can silently rewrite any file already in the chat. The decisive gap is the repository itself: an attacker who controls a repo the user opens can configure commands that run automatically (auto-test, --load, and a further default path), giving unattended code execution with the user's credentials and network.

- **S L2:** Shell commands and URL fetches always need explicit human approval regardless of what was read, but in-chat file edits happen unattended. — [aider/coders/base_coder.py:2456-2462](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L2456-L2462); [aider/coders/base_coder.py:964-981](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L964-L981); [aider/coders/base_coder.py:2198-2200](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L2198-L2200) (verified)
  - *To reach the next level:* Approval is not tied to provenance and edits are not gated once untrusted content has been read.
- **C L1:** The approval gate covers model-proposed actions from any source, but untrusted content enters as user-role text and repo-controlled config escapes the gate entirely. — [aider/coders/base_coder.py:1610-1613](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L1610-L1613); [aider/commands.py:243-244](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/commands.py#L243-L244); [aider/coders/base_prompts.py:26](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_prompts.py#L26); [aider/main.py:464-476](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L464-L476) (verified)
  - *To reach the next level:* Untrusted repo config and content are not treated as data; tool/command output is injected with user standing.
- **D L1:** The gate is on by default, but content in the opened repo (.aider.conf.yml/.env) can switch prompts to yes-always or configure unprompted commands. — [aider/args.py:760-764](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/args.py#L760-L764); [aider/main.py:361-381](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L361-L381); [aider/args.py:35-42](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/args.py#L35-L42) (verified)
  - *To reach the next level:* Repo content must not be able to configure the gate away.
- **B L0:** An attacker-controlled repo can run arbitrary commands unattended (auto-test/test-cmd, --load, and a further default path), leaking secrets and taking irreversible actions. — [aider/main.py:1123-1124](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L1123-L1124); [aider/coders/base_coder.py:1616-1617](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L1616-L1617) (verified)
  - *To reach the next level:* Repo-supplied configuration and content must not run code without a human decision.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.
- **Notes:** One default execution path is not confined.

### C6 Memory, context & configuration integrity — 0.17 (high)

Aider has no long-term memory and does not reload past chats unless asked, but it automatically loads configuration from the repository being edited: .aider.conf.yml, .env (overriding existing environment variables), and model settings files, with no trust prompt. These files can set commands that run automatically, auto-approve prompts, redirect the API endpoint (sending the user's API key to an attacker), or disable TLS verification. The model cannot easily write these files itself because new files and files outside the chat need confirmation and .aider* is gitignored after first run, but a cloned repository can ship them.

- **S L0:** Repo-controlled .aider.conf.yml, .env and model-settings files are loaded silently and can add commands, auto-approve prompts and change API endpoints. — [aider/main.py:464-476](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L464-L476); [aider/main.py:361-381](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L361-L381); [aider/main.py:335-338](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L335-L338); searched `rg -n -i 'trust'` in `aider/main.py aider/args.py` → 0 hits (no workspace-trust decision guards repo-level config loading) (verified)
  - *To reach the next level:* Security-relevant settings from the workspace need an explicit trust decision.
- **C L1:** Model writes to persistent config go through the edit gate (new/out-of-chat files prompt, gitignored files skipped), but no load path is controlled. — [aider/coders/base_coder.py:2206-2209](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L2206-L2209); [aider/coders/base_coder.py:2226-2229](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L2226-L2229); [aider/coders/base_coder.py:2201-2204](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L2201-L2204) (verified)
  - *To reach the next level:* Auto-loaded config, .env and model settings files are uncontrolled.
- **D L1:** Single-user local tool; the repo's own config is applied by default and can change endpoints and approvals. — [aider/args.py:35-42](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/args.py#L35-L42); [aider/args.py:76-79](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/args.py#L76-L79) (verified)
  - *To reach the next level:* Repo-scoped config should not be able to change security settings.
- **B L1:** A planted config persists across every session in that repo and can trigger command execution. — [aider/main.py:1123-1124](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L1123-L1124); [aider/coders/base_coder.py:1616-1617](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L1616-L1617); [aider/args.py:290-294](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/args.py#L290-L294) (verified)
  - *To reach the next level:* Poisoned config persists and can trigger tool use without review.
- **Cap:** C6-REPOCONFIG — Workspace .aider.conf.yml/.env are auto-loaded without a trust decision and can enable commands (test-cmd, lint-cmd, load), auto-approve prompts (yes-always), and redirect the model API base URL.

### C7 Third-party extensions — 1.00 (high)

Aider has no plugin, skill or MCP system and never loads tools or code chosen by the model or by the repository. The only runtime installs are aider's own optional extras (help, browser, Playwright, provider SDKs), pinned in its requirements files, installed only after a 'Run pip install?' prompt showing the exact command. These are the project's own dependencies and are out of scope here, so this criterion's surface is treated as absent.

- **Structural absence:** searched `rg -n -i 'mcp|plugin|entry_points|trust_remote_code|pickle|torch\.load' --glob '*.py'` in `aider` → 0 hits (no extension loading, remote-code model loading or unsafe deserialization in the package); searched `rg -n 'importlib.import_module'` in `aider` → 1 hits (single hit is the lazy import of litellm (llm.py:37), a fixed dependency)
- **Notes:** On-demand pip installs (utils.check_pip_install_extra) install aider's own pinned extras behind a confirmation; with yes-always set (including by repo config) they run unprompted. /help downloads the BAAI/bge-small-en-v1.5 embedding model via llama_index with default (non-remote-code) loading.

### C8 Secrets & sensitive-data protection — 0.30 (high)

API keys come from environment variables, .env files or command-line flags, and are placed into the process environment where every command aider runs can read them. Only the OpenAI and Anthropic keys are masked, and only in the echoed command line and settings dump. The OpenRouter key obtained via OAuth is appended in plain text to ~/.aider/oauth-keys.env without restricting file permissions. Analytics are off unless the user accepts a prompt, but a full unredacted transcript is written to the repository folder by default.

- **S L1:** Keys come from env/.env/flags; masking covers only OpenAI/Anthropic keys in the command-line/settings echo; OAuth key stored plaintext without chmod. — [aider/format_settings.py:1-8](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/format_settings.py#L1-L8); [aider/onboarding.py:361-367](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/onboarding.py#L361-L367) (verified)
  - *To reach the next level:* No type-level masking or log redaction, and stored credentials lack restrictive permissions.
- **C L1:** One path (command line and settings output) is scrubbed; transcripts, subprocess environments and model-bound messages are not. — [aider/main.py:749-751](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L749-L751); [aider/io.py:1128-1136](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/io.py#L1128-L1136); [aider/run_cmd.py:62-72](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/run_cmd.py#L62-L72) (verified)
  - *To reach the next level:* Transcripts and subprocess environments are not protected.
- **D L2:** Analytics need an explicit opt-in prompt (shown to 10% of users) and LLM history logging is off, but the full chat transcript is written unredacted to the repo by default. — [aider/analytics.py:120-135](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/analytics.py#L120-L135); [aider/analytics.py:103-108](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/analytics.py#L103-L108); [aider/args.py:274-288](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/args.py#L274-L288) (verified)
  - *To reach the next level:* Transcripts are not minimised and opted-in analytics enable exception autocapture.
- **B L1:** Long-lived LLM provider keys are reachable by every subprocess. — [aider/main.py:612-616](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/main.py#L612-L616); [aider/run_cmd.py:62-72](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/run_cmd.py#L62-L72) (verified)
  - *To reach the next level:* Keys are long-lived and not short-lived/rotatable by design.
- **Cap:** none

### C9 Audit & traceability — 0.38 (high)

By default aider appends a Markdown transcript of each session (user input, model replies, commands run, confirmation answers) to .aider.chat.history.md in the repository, and every AI edit becomes a git commit tagged with a 'Co-authored-by: aider' trailer. This gives a usable record, but it is unstructured, lives inside the workspace where it can be edited, and if writing fails aider prints a warning and continues without logging.

- **S L1:** Unstructured Markdown transcript plus git commits with an aider trailer. — [aider/io.py:1128-1136](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/io.py#L1128-L1136); [aider/repo.py:252](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/repo.py#L252) (verified)
  - *To reach the next level:* No structured per-action record with arguments, status and timestamps (the opt-in LLM history file is also free text).
- **C L2:** Edits, commands, outputs and confirmation answers all flow through the io layer into the transcript. — [aider/io.py:920-923](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/io.py#L920-L923); [aider/io.py:336](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/io.py#L336) (verified)
  - *To reach the next level:* No record of configuration loaded or of commands run from --load/test-cmd attributed as config-driven.
- **D L1:** On by default but stored in the repo root, where the agent's own edits and shell commands can alter it. — [aider/args.py:274-288](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/args.py#L274-L288) (verified)
  - *To reach the next level:* Store the record outside the workspace.
- **B L2:** Each entry is appended and closed per write; write failures are printed and logging is then disabled while actions continue. — [aider/io.py:1128-1136](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/io.py#L1128-L1136) (verified)
  - *To reach the next level:* Records are not replayable and logging failure does not stop actions.
- **Cap:** none

### C10 Limits & kill switch — 0.42 (high)

Each user message triggers at most one model reply plus three automatic retries for lint, edit or file-add follow-ups, and each model request has a 10-minute timeout, so aider is human-paced by design. There is no spend limit (cost is only displayed), no session time limit, and commands run with no timeout. Ctrl-C interrupts the model reply and a second Ctrl-C exits; a running command receives the interrupt through the terminal.

- **S L2:** Hard-coded 3-reflection cap per message plus a 600s per-request timeout, enforced in code. — [aider/coders/base_coder.py:101](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L101); [aider/coders/base_coder.py:939-943](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L939-L943); [aider/models.py:28](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/models.py#L28); searched `rg -n -i 'max_cost|cost_limit|spend_limit' --glob '*.py'` in `aider` → 0 hits (no spend cap; cost is only displayed) (verified)
  - *To reach the next level:* No wall-clock or cost cap and no rate limits on side-effecting actions.
- **C L1:** Limits cover the top-level loop and model calls; shell, lint and test commands have no timeout, and the architect editor sub-coder gets its own reflection budget. — searched `rg -n 'timeout'` in `aider/run_cmd.py` → 0 hits (shell/test/lint commands run with no timeout); [aider/coders/architect_coder.py:24-33](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/architect_coder.py#L24-L33) (verified)
  - *To reach the next level:* Command executions need timeouts.
- **D L2:** Sensible hard-coded defaults the model cannot change, but delegation to the architect editor starts a fresh reflection count. — [aider/coders/base_coder.py:101](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L101); [aider/coders/architect_coder.py:24-33](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/architect_coder.py#L24-L33) (verified)
  - *To reach the next level:* Delegated sub-coders should share the parent's reflection budget.
- **B L2:** Human-paced turns with small per-message ceilings; Ctrl-C stops the loop, but no spend ceiling and commands may run indefinitely. — [aider/coders/base_coder.py:986-996](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L986-L996); searched `rg -n -i 'max_cost|cost_limit|spend_limit' --glob '*.py'` in `aider` → 0 hits (no spend cap; cost is only displayed) (verified)
  - *To reach the next level:* No spend ceiling and no guaranteed cancellation of running commands.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Repo files/repo map and repo config (aider/main.py:473), web pages (aider/commands.py:244), command output as user messages (aider/coders/base_coder.py:1612) · [B] sensitive data/systems: LLM keys and the user's environment inherited by all subprocesses (aider/main.py:616, aider/run_cmd.py:62) · [C] state change / egress: Unprompted edits to in-chat files (aider/coders/base_coder.py:2198), approved shell commands (aider/coders/base_coder.py:2475), config-driven test/load commands (aider/main.py:1124) · Same default session? Yes

## Highest-impact improvements
1. Require an explicit workspace-trust decision before applying .aider.conf.yml, .env and model-settings files from the repo, or ignore security-relevant keys (commands, yes-always, API base, verify-ssl, load) from repo scope. — C6 S L0→L3, +0.225 before caps (Playbook 2)
2. Require approval for configured test/lint/load commands that come from repo config, and harden default execution paths. — C2 D L1→L3, +0.100 before caps (Playbook 5)
3. Pass a scrubbed environment (no provider keys or cloud credentials) to shell, lint and test subprocesses. — C1 C L0→L1, +0.075 before caps (Playbook 4)
4. Add timeouts to run_cmd and a configurable per-session spend cap. — C10 C L1→L2, +0.075 before caps (Playbook 3)
5. Write OAuth keys with 0600 permissions and redact all provider keys from transcripts. — C8 S L1→L2, +0.075 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- One execution path is inferred and was not executed.
- Benchmark scripts, the website, the Streamlit GUI (--gui), watch-files mode and the voice feature were not reviewed in depth; architect mode was considered only for limits.
- The opt-in Docker image was scored as the C4 alternative from the Dockerfile and install docs only.
- C7 was treated as structurally absent after re-checking runtime pip installs (aider's own pinned extras behind a consent prompt); if those are counted as extensions the criterion would score far lower.
- No text aimed at AI reviewers was found in README, CONTRIBUTING or prompt files.
