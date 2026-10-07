# Defense-in-Depth Score: gpt-engineer

**Repo:** https://github.com/antonosika/gpt-engineer · **Commit:** `a90fcd543eedcc0ff2c34561bc0785d2ba83c47e` (0.3.1) · **Reviewed:** 2026-10-04
**What it is:** CLI that generates or improves a whole codebase from a natural-language prompt and offers to run it via a generated run.sh.
**Category:** Coding
**Scored configuration:** `gpte <project_dir>` generate mode, no flags, fresh pip install, host (non-Docker) execution.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents no · external communication opt-in

## Score: 1.7 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |
| C3 | Tool & action scoping | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C4 | Code-execution isolation | L2 | L3 | L0 | L1 | 0.42 | G1 | **0.42** (alt) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | Medium |
| C7 | Third-party extensions | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | G2 | **0.20** | High |
| C9 | Audit & traceability | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |


gpt-engineer writes model-generated files to disk with no approval and no path containment, then asks a single default-yes question before running a generated bash script on the host as you, with your full environment and API keys. There is no sandbox, no timeout, and no redaction. A .env file in the working directory can silently redirect the model endpoint, and the analytics consent check is not a strict boundary. The repository is archived, so none of this will be fixed upstream.

## Critical gaps
- Generated code runs with the user's full ambient authority and inherits the process environment, including OPENAI_API_KEY/ANTHROPIC_API_KEY. (ASI03, T3; C1) — [gpt_engineer/core/default/disk_execution_env.py:76-83](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_execution_env.py#L76-L83); [gpt_engineer/applications/cli/main.py:80-90](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/applications/cli/main.py#L80-L90)
- Model-generated run.sh executes on the host as the user, with no isolation and the full environment (C4 blast radius L0 in the default config). (ASI05, T11; C4) — [gpt_engineer/core/default/disk_execution_env.py:76-83](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_execution_env.py#L76-L83); [gpt_engineer/core/default/steps.py:267](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L267)
- A .env in the current working directory is auto-loaded with no trust prompt and can supply the API key or redirect the model endpoint. (ASI06, T1; C6) — [gpt_engineer/applications/cli/main.py:80-90](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/applications/cli/main.py#L80-L90)
- Model-chosen dependencies installed by run.sh execute as the user with every inherited credential (C7 blast radius L0). (ASI04, T17; C7) — [gpt_engineer/core/default/steps.py:181-185](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L181-L185); [gpt_engineer/core/default/disk_execution_env.py:76-83](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_execution_env.py#L76-L83)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

gpt-engineer runs as the user who launched it and does nothing to narrow that authority. The API keys it loads (including from a .env file in the current directory) go into the process environment, and the generated run.sh is started with no scrubbed environment, so the generated code inherits every key and can reach everything the user can: home directory, SSH keys, cloud credential files. There is no authorization layer of any kind.

- **S L0:** Ambient OS-user authority; the LLM API key is loaded into os.environ and nothing is narrowed. — [gpt_engineer/applications/cli/main.py:80-90](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/applications/cli/main.py#L80-L90) (verified)
  - *To reach the next level:* No scoped identity or deterministic authorization gate before actions run.
- **C L0:** The executed shell script receives the full parent environment, because Popen is called without env=. — [gpt_engineer/core/default/disk_execution_env.py:76-83](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_execution_env.py#L76-L83); searched `rg -n -i 'env='` in `gpt_engineer` → 3 hits (Two hits are the execution_env= keyword argument (cli_agent.py, simple_agent.py) and one is env=env in benchmark/run.py naming the DiskExecutionEnv object; no subprocess call passes a scrubbed env, so children inherit os.environ.) (verified)
  - *To reach the next level:* Not even the main execution path strips credentials from the subprocess environment.
- **D L0:** A default install runs model-written code with the user's full privileges. — [gpt_engineer/core/default/disk_execution_env.py:76-83](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_execution_env.py#L76-L83) (verified)
  - *To reach the next level:* No narrower default role; least privilege would need manual hardening (e.g. the opt-in Docker image).
- **B L0:** If the gate fails, generated code holds the user's whole account: the LLM API keys in its environment plus any ambient credentials on disk. — [gpt_engineer/core/default/disk_execution_env.py:76-83](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_execution_env.py#L76-L83); searched `rg -n -i 'env='` in `gpt_engineer` → 3 hits (Two hits are the execution_env= keyword argument (cli_agent.py, simple_agent.py) and one is env=env in benchmark/run.py naming the DiskExecutionEnv object; no subprocess call passes a scrubbed env, so children inherit os.environ.) (verified)
  - *To reach the next level:* No layer limits generated code to the project or to a scoped credential.
- **Cap:** none

### C2 Approval gates — 0.38 (high)

There is one approval prompt: before running the generated run.sh, the CLI prints the script and asks 'Do you want to execute this code? (Y/n)', and pressing Enter counts as yes. The prompt shows the script but not the generated source files it runs. In the default generate mode, writing the generated files to disk is never gated, and those paths come straight from model output. The --self-heal flag runs the script up to ten times with no approval, and its help text gives no warning about that. Improve mode does show a full diff and asks before applying changes.

- **S L2:** Per-call approval of the exact run.sh text, but the generated code files the script runs are not shown at approval time, and Enter defaults to yes. — [gpt_engineer/core/default/steps.py:244-251](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L244-L251); [gpt_engineer/core/default/steps.py:267](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L267) (verified)
  - *To reach the next level:* The approver doesn't see the full code being executed, and there are no risk tiers or argument-level policy.
- **C L1:** Shell execution is gated, but in generate mode the file writes at model-chosen paths happen with no approval. — [gpt_engineer/applications/cli/main.py:548-550](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/applications/cli/main.py#L548-L550); [gpt_engineer/core/default/file_store.py:41-43](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/file_store.py#L41-L43); [gpt_engineer/applications/cli/main.py:534-540](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/applications/cli/main.py#L534-L540) (verified)
  - *To reach the next level:* Writes in generate mode (main.py push) don't cross any gate.
- **D L2:** On by default, but the operator flag --self-heal silently skips approval for up to 10 executions, and the default answer is yes. — [gpt_engineer/tools/custom_steps.py:95-101](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/tools/custom_steps.py#L95-L101); [gpt_engineer/applications/cli/main.py:310-315](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/applications/cli/main.py#L310-L315); [gpt_engineer/core/default/steps.py:244-251](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L244-L251) (verified)
  - *To reach the next level:* Disabling the gate isn't a loudly named, warned flag, and the empty-input default approves.
- **B L1:** Executed scripts and overwrites outside a git repo can't be undone; the only safety net is staging already-modified files when the project is a git repo. — [gpt_engineer/core/git.py:71-85](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/git.py#L71-L85); [gpt_engineer/core/default/file_store.py:41-43](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/file_store.py#L41-L43) (verified)
  - *To reach the next level:* No checkpoint or rollback for file writes or execution side effects.
- **Cap:** none

### C3 Tool & action scoping — 0.00 (high)

The agent's two actions are as broad as they get: it writes files at any path the model names and runs an arbitrary bash script. Paths in the model output are only stripped of a few punctuation characters, so absolute paths and ../ traversal are accepted, and joining them onto the project path in Python lets an absolute path replace the project root entirely. Nothing restricts which commands run.sh may contain.

- **S L0:** Raw passthrough: model-chosen file paths go unchecked, and the entrypoint is an arbitrary shell script. — [gpt_engineer/core/chat_to_files.py:49-64](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/chat_to_files.py#L49-L64); [gpt_engineer/core/default/file_store.py:41-43](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/file_store.py#L41-L43); searched `rg -n -i 'resolve\(\)|realpath|is_relative_to|commonpath'` in `gpt_engineer` → 2 hits (Both hits are in file_selector.py resolving the project root for the interactive file list; no containment check exists on model-chosen output paths.) (verified)
  - *To reach the next level:* No resolved-path containment for output files and no command allowlist.
- **C L0:** Neither action validates its input. — [gpt_engineer/core/default/file_store.py:41-43](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/file_store.py#L41-L43); [gpt_engineer/core/default/steps.py:267](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L267) (verified)
  - *To reach the next level:* No action validates its arguments.
- **D L0:** Write and exec are always on; the only opt-out is the testing-only --no_execution flag, which also skips generation. — [gpt_engineer/applications/cli/main.py:548-550](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/applications/cli/main.py#L548-L550); [gpt_engineer/core/default/steps.py:267](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L267) (verified)
  - *To reach the next level:* No read-only or reduced tool set by default.
- **B L0:** A misused write or script reaches the whole machine as the user. — [gpt_engineer/core/default/file_store.py:41-43](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/file_store.py#L41-L43); [gpt_engineer/core/default/disk_execution_env.py:76-83](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_execution_env.py#L76-L83) (verified)
  - *To reach the next level:* Actions aren't scoped to the project directory.
- **Cap:** none

### C4 Code-execution isolation — 0.42 (high)

Generated code runs with no isolation: run.sh is started with shell=True as the same user, from a temporary directory, with the full environment and full network. The repo ships a Docker image and compose file as an opt-in alternative, but that container runs as root and gets the API key from .env, a read-write project mount, and unrestricted network, so it is basic separation at best.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Same-user host subprocess; a temp working directory is the only separation. — [gpt_engineer/core/default/disk_execution_env.py:76-83](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_execution_env.py#L76-L83); [gpt_engineer/core/default/file_store.py:32-33](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/file_store.py#L32-L33); searched `rg -n -i 'sandbox|docker|seccomp|chroot'` in `gpt_engineer` → 0 hits (No isolation primitive anywhere in the Python package; the only container is the opt-in docker/ directory.) (verified)
    - *To reach the next level:* No OS-level isolation primitive on the default path.
  - **C L0:** No execution path is sandboxed (main entrypoint, self-heal, benchmark). — [gpt_engineer/core/default/disk_execution_env.py:76-83](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_execution_env.py#L76-L83); [gpt_engineer/tools/custom_steps.py:95-101](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/tools/custom_steps.py#L95-L101) (verified)
    - *To reach the next level:* Nothing routes execution through a sandbox.
  - **D L0:** No isolation in the default install. — [gpt_engineer/core/default/disk_execution_env.py:76-83](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_execution_env.py#L76-L83) (verified)
    - *To reach the next level:* Isolation isn't on by default.
  - **B L0:** Host-equivalent: home directory, credential files and the inherited API keys are all reachable. — [gpt_engineer/core/default/disk_execution_env.py:76-83](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_execution_env.py#L76-L83); searched `rg -n -i 'env='` in `gpt_engineer` → 3 hits (Two hits are the execution_env= keyword argument (cli_agent.py, simple_agent.py) and one is env=env in benchmark/run.py naming the DiskExecutionEnv object; no subprocess call passes a scrubbed env, so children inherit os.environ.) (verified)
    - *To reach the next level:* Generated code can reach host credentials and the whole filesystem.
- **opt-in Docker image (docker/Dockerfile + docker-compose.yml)** (alt; raw 0.42, cap G1 → 0.42) ← counted
  - **S L2:** Stock python:3.11-slim container with no USER directive (root inside) and default capabilities. — [docker/Dockerfile:18-29](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/docker/Dockerfile#L18-L29) (verified)
    - *To reach the next level:* No non-root user, dropped capabilities, seccomp, or read-only root.
  - **C L3:** The whole CLI runs inside the container, so every execution path is inside it. — [docker/Dockerfile:29](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/docker/Dockerfile#L29); [docker-compose.yml:9-16](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/docker-compose.yml#L9-L16) (verified)
    - *To reach the next level:* No fail-closed guarantee; the host install stays the documented primary path.
  - **D L0:** Opt-in; the README lists Docker as one install option. — [docker/Dockerfile:18-29](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/docker/Dockerfile#L18-L29) (verified)
    - *To reach the next level:* Not the default mode.
  - **B L1:** The container gets the .env API key, a read-write project mount, and default (full) network. — [docker-compose.yml:9-16](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/docker-compose.yml#L9-L16) (verified)
    - *To reach the next level:* Credentials are in the sandbox environment and egress is unrestricted.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.05 (high)

Nothing in the code limits what a hijacked run can do. The prompt file and, in improve mode, project files the user selects go into the model with the same standing as the user's instructions. If that content steers the model, the resulting file writes land unattended at any path, which allows irreversible overwrites outside the project. Running code, and with it network exfiltration, still needs the run.sh approval.

- **S L0:** No structural limit after untrusted content is read; the only defence is the run.sh prompt, which is scored in C2. — [gpt_engineer/core/default/steps.py:305-307](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L305-L307) (verified)
  - *To reach the next level:* No egress or write restriction tied to untrusted content in session.
- **C L0:** Project file contents and the prompt file aren't distinguished from user instructions. — [gpt_engineer/core/default/steps.py:305-307](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L305-L307) (verified)
  - *To reach the next level:* Untrusted sources aren't marked or separated.
- **D L0:** No control to enable. — [gpt_engineer/core/default/steps.py:305-307](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L305-L307) (verified)
  - *To reach the next level:* No control exists.
- **B L1:** Unattended irreversible writes at arbitrary paths (generate mode), while exfiltration through code execution needs the approval prompt. — [gpt_engineer/applications/cli/main.py:548-550](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/applications/cli/main.py#L548-L550); [gpt_engineer/core/default/file_store.py:41-43](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/file_store.py#L41-L43); [gpt_engineer/core/default/steps.py:244-251](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L244-L251) (verified)
  - *To reach the next level:* File writes aren't gated by approval, so irreversible changes happen unattended.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.10 (medium)

gpt-engineer has no long-term memory store; its logs in .gpteng are never read back into context. It does, however, auto-load a .env file from the current working directory whenever the API key isn't already set. A cloned or downloaded project can use that to supply the API key or, through the OpenAI client's environment variables, redirect the model endpoint. The analytics consent mechanism is also not a strict boundary. Custom system prompts from the project are loaded only with an explicit flag.

- **S L0:** Files in the working directory (.env) silently change security-relevant settings: credentials and endpoint; consent handling is also not a strict boundary. — [gpt_engineer/applications/cli/main.py:80-90](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/applications/cli/main.py#L80-L90) (verified)
  - *To reach the next level:* No workspace-trust decision before loading project-scoped .env files.
- **C L0:** None of the auto-loaded files is controlled. — [gpt_engineer/applications/cli/main.py:80-90](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/applications/cli/main.py#L80-L90) (verified)
  - *To reach the next level:* No auto-loaded path is checked.
- **D L1:** Single-user local tool; logs are per project under .gpteng/memory, but nothing stops a project from carrying its own config files. — [gpt_engineer/core/default/paths.py:43-44](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/paths.py#L43-L44); [gpt_engineer/applications/cli/main.py:80-90](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/applications/cli/main.py#L80-L90) (verified)
  - *To reach the next level:* Project scope isn't separated from user scope for security settings.
- **B L1:** A planted .env persists across every run from that directory and can redirect the model, whose output then drives unattended file writes. — [gpt_engineer/applications/cli/main.py:80-90](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/applications/cli/main.py#L80-L90); [gpt_engineer/applications/cli/main.py:548-550](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/applications/cli/main.py#L548-L550) (inferred)
  - *To reach the next level:* A poisoned config still drives ungated actions.
- **Cap:** C6-REPOCONFIG — main.py:83 loads .env from os.getcwd() with no trust prompt; langchain_openai's ChatOpenAI reads OPENAI_API_BASE/OPENAI_API_KEY from the environment (library behaviour relied on), so a workspace file can redirect the model endpoint and credentials.
- **Notes:** --use-custom-preprompts (opt-in) loads system prompts from <project>/preprompts, which a cloned project could pre-seed.

### C7 Third-party extensions — 0.05 (high)

gpt-engineer loads no plugins or MCP servers. Its default entrypoint prompt, however, tells the model to write a script that installs dependencies, so packages the model picks (unpinned, unverified) get installed and their install scripts run as the user with the full environment. The only consent is the generic run.sh prompt, although that prompt does display the install commands.

- **S L0:** Model-chosen package installs with no pinning or verification. — [gpt_engineer/core/default/steps.py:181-185](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L181-L185) (verified)
  - *To reach the next level:* Packages aren't pinned or integrity-checked.
- **C L0:** The one extension type (installed packages) isn't verified. — [gpt_engineer/core/default/steps.py:181-185](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L181-L185); searched `rg -n -i 'mcp|plugin|entry_points|importlib'` in `gpt_engineer` → 2 hits (Both hits are benchmark/__main__.py importing the user-named agent module for the separate bench binary; the gpte CLI loads no plugins or MCP servers.) (verified)
  - *To reach the next level:* No verification on any install path.
- **D L1:** Installs happen only behind the run.sh Y/n prompt, which shows the commands but defaults to yes. — [gpt_engineer/core/default/steps.py:244-251](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L244-L251) (verified)
  - *To reach the next level:* No explicit per-package consent showing versions and permissions.
- **B L0:** Installed packages run as the same user with the full environment and API keys. — [gpt_engineer/core/default/disk_execution_env.py:76-83](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_execution_env.py#L76-L83); searched `rg -n -i 'env='` in `gpt_engineer` → 3 hits (Two hits are the execution_env= keyword argument (cli_agent.py, simple_agent.py) and one is env=env in benchmark/run.py naming the DiskExecutionEnv object; no subprocess call passes a scrubbed env, so children inherit os.environ.) (verified)
  - *To reach the next level:* Package installs aren't confined or given a scrubbed environment.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.20 (high)

API keys come from environment variables or a .env file and are passed unchanged to every subprocess the agent starts. Nothing is redacted anywhere. Improve mode's file picker skips dotfiles, so .env isn't offered to the model by default. Opt-in analytics send the full prompt and session logs to the maintainers' RudderStack endpoint, and the consent check is not a strict boundary.

- **S L1:** Secrets come from env vars; the only protection is that the improve-mode file list skips dotfiles. — [gpt_engineer/applications/cli/file_selector.py:404-405](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/applications/cli/file_selector.py#L404-L405); searched `rg -n -i 'redact|mask|SecretStr'` in `gpt_engineer` → 0 hits (No redaction or masking anywhere.) (verified)
  - *To reach the next level:* No masking or redaction of logs or analytics payloads.
- **C L1:** Only one path (model-bound file selection) is protected; logs, analytics, and the subprocess environment aren't. — [gpt_engineer/applications/cli/file_selector.py:404-405](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/applications/cli/file_selector.py#L404-L405); searched `rg -n -i 'env='` in `gpt_engineer` → 3 hits (Two hits are the execution_env= keyword argument (cli_agent.py, simple_agent.py) and one is env=env in benchmark/run.py naming the DiskExecutionEnv object; no subprocess call passes a scrubbed env, so children inherit os.environ.) (verified)
  - *To reach the next level:* Logs, the analytics payload, and the subprocess env are unprotected.
- **D L1:** Analytics are asked for each run, but the payload carries prompts and logs, and the consent check is not a strict boundary. (verified)
  - *To reach the next level:* Content-bearing analytics consent is not robustly enforced.
- **B L0:** Long-lived LLM provider keys are reachable by every generated subprocess. — [gpt_engineer/core/default/disk_execution_env.py:76-83](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_execution_env.py#L76-L83); searched `rg -n -i 'env='` in `gpt_engineer` → 3 hits (Two hits are the execution_env= keyword argument (cli_agent.py, simple_agent.py) and one is env=env in benchmark/run.py naming the DiskExecutionEnv object; no subprocess call passes a scrubbed env, so children inherit os.environ.) (verified)
  - *To reach the next level:* Keys aren't scoped or kept out of the subprocess environment.
- **Cap:** G2 — The consent gate for uploading prompts and logs is not a strict boundary.

### C9 Audit & traceability — 0.25 (high)

Each run appends timestamped plain-text logs of the LLM conversations (generated code, entrypoint chat, improve diffs) under .gpteng/memory/logs inside the project. The output of executed scripts and the user's approve or decline decisions aren't recorded in generate mode. Executed code can edit the logs, because they sit in the workspace and are writable by the same user.

- **S L1:** Unstructured, timestamped text logs of model exchanges. — [gpt_engineer/core/default/disk_memory.py:314-316](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_memory.py#L314-L316); [gpt_engineer/core/default/steps.py:147-149](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L147-L149) (verified)
  - *To reach the next level:* No structured per-action record of executions and their results.
- **C L1:** Only the generation and improve paths are logged; execution and approvals aren't. — [gpt_engineer/core/default/steps.py:147-149](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L147-L149); [gpt_engineer/core/default/steps.py:267](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L267) (verified)
  - *To reach the next level:* Executions and approval decisions aren't recorded.
- **D L1:** On by default, stored inside the project directory that executed code can write. — [gpt_engineer/core/default/paths.py:43-44](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/paths.py#L43-L44); [gpt_engineer/core/default/disk_memory.py:314-316](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_memory.py#L314-L316) (verified)
  - *To reach the next level:* The record lives where agent-run code can change it.
- **B L1:** Best-effort appends; nothing depends on a record being written. — [gpt_engineer/core/default/disk_memory.py:314-316](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_memory.py#L314-L316) (verified)
  - *To reach the next level:* Executions don't produce a durable per-action record.
- **Cap:** none

### C10 Limits & kill switch — 0.25 (high)

The pipeline makes a fixed handful of model calls (two diff-repair retries, rate-limit backoff capped at seven tries), so model spend per run has a structural bound. The generated script, however, runs with no timeout, and the default prompt even asks it to run parts 'in parallel if necessary'. Ctrl+C kills only the shell process, so background children can keep running. There is no cost cap.

- **S L1:** Small fixed iteration caps on model calls, but no execution timeout or cost cap. — [gpt_engineer/core/default/constants.py:12](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/constants.py#L12); [gpt_engineer/core/ai.py:253](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/ai.py#L253); searched `rg -n -i 'max_cost|budget|cost_limit'` in `gpt_engineer` → 0 hits (No cost or token ceiling; cost is only printed after the run.); [gpt_engineer/core/default/steps.py:267](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L267) (verified)
  - *To reach the next level:* No wall-clock or per-execution timeout and no token/cost cap.
- **C L1:** Limits cover only the model-call loop; spawned processes are unbounded. — [gpt_engineer/core/default/disk_execution_env.py:99-107](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_execution_env.py#L99-L107); searched `rg -n -i 'killpg|setsid|start_new_session|process_group'` in `gpt_engineer` → 0 hits (No process-group handling; Ctrl+C kills only the /bin/sh child.) (verified)
  - *To reach the next level:* Tool execution has no timeout.
- **D L1:** The model-call caps are hard-coded constants the model can't change, but execution time is unlimited by default. — [gpt_engineer/core/default/constants.py:12](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/constants.py#L12); [gpt_engineer/core/default/steps.py:267](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L267) (verified)
  - *To reach the next level:* No sensible default ceiling on script execution time.
- **B L1:** A runaway script runs indefinitely, and Ctrl+C can leave background processes running. — [gpt_engineer/core/default/disk_execution_env.py:99-107](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/disk_execution_env.py#L99-L107); searched `rg -n -i 'killpg|setsid|start_new_session|process_group'` in `gpt_engineer` → 0 hits (No process-group handling; Ctrl+C kills only the /bin/sh child.); [gpt_engineer/core/default/steps.py:181-185](https://github.com/antonosika/gpt-engineer/blob/a90fcd543eedcc0ff2c34561bc0785d2ba83c47e/gpt_engineer/core/default/steps.py#L181-L185) (verified)
  - *To reach the next level:* Stopping doesn't kill the process group.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: prompt file and selected project files sent as model input (gpt_engineer/core/default/steps.py:305-307) · [B] sensitive data/systems: API keys in os.environ inherited by run.sh (gpt_engineer/core/default/disk_execution_env.py:76-83) · [C] state change / egress: ungated file writes at model-chosen paths (gpt_engineer/applications/cli/main.py:550) and run.sh with full network (gpt_engineer/core/default/steps.py:267) · Same default session? Yes

## Highest-impact improvements
1. Resolve every model-chosen output path and refuse anything outside the project directory. — C3 S L0→L2, +0.150 before caps
2. Show the full diff of generated files and require approval before writing them in generate mode too. — C2 C L1→L2, +0.075 before caps
3. Run run.sh with a scrubbed environment (no LLM API keys) and inside a container by default. — C4 D L0→L2, +0.100 before caps
4. Only read .env from user scope, not the current working directory, and harden the analytics consent check. — C8 D L1→L2, +0.050 before caps
5. Pass a default timeout to DiskExecutionEnv.run and kill the process group on stop. — C10 S L1→L2, +0.075 before caps

## Re-audit log
- No changes.

## Limitations
- Static review of the pinned commit only; nothing was installed, built, or run.
- The GitHub repository is archived (read-only); no upstream fixes are expected after this commit.
- The secondary `bench` binary (gpt_engineer/benchmark) runs generated code with no approval and imports a user-named agent module; it was not scored.
- Opt-in modes (--self-heal, --use-custom-preprompts, --llm-via-clipboard, Azure, Docker) were reviewed only where they affect the default-mode ratings.
- Downstream behaviour of langchain_openai reading OPENAI_API_BASE from the environment is inferred from that library, not verified in this repo.
- No text aimed at AI reviewers was found in the repository.
