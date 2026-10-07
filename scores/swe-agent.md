# Defense-in-Depth Score: SWE-agent

**Repo:** https://github.com/SWE-agent/SWE-agent · **Commit:** `3ea751c087f32b16e039a2233dd6eefecef325d5` · **Reviewed:** 2026-10-03
**What it is:** Takes a GitHub issue and autonomously fixes it with an LM
**Category:** Coding
**Scored configuration:** `sweagent run` with the default config/default.yaml on a GitHub issue URL: default Docker (python:3.11) deployment via SWE-ReX, bash + str_replace_editor + submit tools, $3 per-instance cost cap, open_pr and apply_patch_locally off, GITHUB_TOKEN present in the environment or .env as the shipped .env.example asks.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents no · external communication opt-in

## Score: 2.9 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C2 | Approval gates | L0 | L0 | L0 | L2 | 0.10 | — | **0.10** | High |
| C3 | Tool & action scoping | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C4 | Code-execution isolation | L2 | L3 | L2 | L0 | 0.47 | — | **0.47** | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L2 | SA | L3 | 0.57 | — | **0.57** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C9 | Audit & traceability | L2 | L3 | L3 | L3 | 0.68 | — | **0.68** | High |
| C10 | Limits & kill switch | L1 | L2 | L2 | L2 | 0.42 | — | **0.42** | Medium |


SWE-agent runs every model command in a throwaway Docker container, which keeps the host filesystem and the model API key out of reach. But it has no approval step and no egress limits, and it feeds an issue that anyone can write straight into the prompt. GitHub token handling is also not locked down. Prefer running without a token, or with a fine-grained read-only token.

## Critical gaps
- Untrusted GitHub issue text drives an unattended agent with open egress: exfiltration and irreversible actions need no human (C5-WORSTCASE). (ASI01, T6, LLM01; C5) — [sweagent/utils/github.py:103-107](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/utils/github.py#L103-L107); [config/default.yaml:14-16](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/config/default.yaml#L14-L16)

## Criterion details

### C1 Identity & least privilege — 0.05 (high)

SWE-agent keeps the LLM API key in the host process and does not pass the operator's environment into the container, which is a real narrowing. But whenever a GITHUB_TOKEN is set (the shipped .env.example asks for one with full 'repo' scope for private repositories), it is used with no narrowing, and its handling is not locked down. There is no per-request authorization layer and no token downscoping.

- **S L0:** The operator's ambient GitHub personal access token is used with no narrowing, and its handling is not locked down. — [.env.example:5-8](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/.env.example#L5-L8) (verified)
  - *To reach the next level:* No dedicated or narrowed identity: the token is neither scoped nor replaced by a short-lived, repo-specific credential.
- **C L0:** No authorization check sits between model commands and the GitHub credential. — [sweagent/agent/agents.py:946-967](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L946-L967); [sweagent/tools/tools.py:86-92](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/tools/tools.py#L86-L92) (verified)
  - *To reach the next level:* No code-level authorization on the main tool path.
- **D L1:** Without GITHUB_TOKEN no external credentials are involved, but merely having the variable set (or loaded from .env) widens the default with no flag or warning. — [sweagent/utils/config.py:65-78](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/utils/config.py#L65-L78); [.env.example:5-8](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/.env.example#L5-L8) (verified)
  - *To reach the next level:* Credential use should require an explicit opt-in flag and default to read-only or none.
- **B L0:** A hijacked model holding a classic PAT with 'repo' scope can push, rewrite, or open PRs on every repository the user can write, across organizations. — [.env.example:5-8](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/.env.example#L5-L8) (verified)
  - *To reach the next level:* Credential not limited to the one target repository, read-only, or short-lived.
- **Cap:** none

### C2 Approval gates — 0.10 (high)

There is no human approval step anywhere in SWE-agent: every model command, including arbitrary shell, runs immediately inside the container. Most damage is contained because edits happen in a throwaway container and the default output is only a patch file the user reviews. However, network calls from inside the container are unattended and can be irreversible; opening a PR is opt-in and happens without a per-PR confirmation.

- **S L0:** No approval mechanism exists; the loop executes each parsed action directly. — searched `rg -n -i 'approv|confirm'` in `sweagent` → 0 hits (No approval or confirmation logic anywhere in the agent package.); [sweagent/agent/agents.py:946-967](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L946-L967) (verified)
  - *To reach the next level:* No per-call human approval showing the exact command.
- **C L0:** The most powerful tool, bash, is enabled by default and ungated. — [config/default.yaml:64](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/config/default.yaml#L64); [sweagent/tools/tools.py:116](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/tools/tools.py#L116); [sweagent/agent/agents.py:946-967](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L946-L967) (verified)
  - *To reach the next level:* Shell and network-capable commands are not routed through any gate.
- **D L0:** No approval mode exists to be on by default. — searched `rg -n -i 'approv|confirm'` in `sweagent` → 0 hits (No approval or confirmation logic anywhere in the agent package.) (verified)
  - *To reach the next level:* Approval is not available at all, let alone on by default.
- **B L2:** Filesystem changes live in an ephemeral container and the default result is a saved patch, so the common case is reversible; network egress from the container is not. — [sweagent/environment/swe_env.py:27-30](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/environment/swe_env.py#L27-L30); [sweagent/run/run_single.py:59](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/run/run_single.py#L59) (verified)
  - *To reach the next level:* No preview/dry-run or rollback for external actions such as pushes or HTTP calls made from the container.
- **Cap:** none

### C3 Tool & action scoping — 0.38 (high)

The default tool set is a raw bash shell plus a file editor, both with full reach inside the container and unrestricted network. The only argument filter is a short blocklist of interactive programs (vim, less, bare python) matched by string prefix or exact match; it exists to avoid hanging the session, not for safety. Every action passes through that filter, but it is trivially bypassed and nothing validates paths, hosts, or quantities.

- **S L1:** Actions are filtered only by a denylist of interactive commands using startswith/exact matching. — [sweagent/tools/tools.py:37-49](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/tools/tools.py#L37-L49); [sweagent/tools/tools.py:353-367](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/tools/tools.py#L353-L367) (verified)
  - *To reach the next level:* No allowlist validation in code (path containment, host allowlists, bounds) on tool arguments.
- **C L2:** The blocklist check runs on every parsed action before execution, so all tools (which are shell commands) pass through it. — [sweagent/agent/agents.py:946-947](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L946-L947); [sweagent/tools/tools.py:353-367](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/tools/tools.py#L353-L367) (verified)
  - *To reach the next level:* No shared validation layer that enforces argument allowlists across tools.
- **D L2:** Tool groups are selectable (bundles listed in config, bash switchable via enable_bash_tool), but the default group includes bash, file editing and network. — [config/default.yaml:64](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/config/default.yaml#L64); [sweagent/tools/tools.py:116](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/tools/tools.py#L116); [config/default.yaml:41-44](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/config/default.yaml#L41-L44) (verified)
  - *To reach the next level:* The default tool set is not read-only; write and exec are on without explicit enabling.
- **B L1:** A misused tool can run any command as root inside the container and reach any network host, but cannot touch the host filesystem by default. — [sweagent/environment/swe_env.py:27-30](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/environment/swe_env.py#L27-L30); [sweagent/environment/swe_env.py:220-222](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/environment/swe_env.py#L220-L222) (verified)
  - *To reach the next level:* Reach not scoped to the workspace with network limits or quantity bounds.
- **Cap:** none

### C4 Code-execution isolation — 0.47 (medium)

By default every model command runs in a fresh Docker container (python:3.11) through SWE-ReX, and the container is removed after the run. This covers all model-reachable execution paths, including bundled tools that are uploaded and run inside it, and there is no automatic fallback to the host. The container is a stock one: root inside, default capabilities, unrestricted network, and no hardening flags. GitHub token handling for the sandbox is not locked down.

- **S L2:** Isolation is a default Docker container with no docker_args hardening (root, default capabilities, default network) as configured by the repo; the docker run construction lives in the SWE-ReX library. — [sweagent/environment/swe_env.py:27-30](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/environment/swe_env.py#L27-L30); searched `rg -n -e 'docker_args' -e 'cap-drop' -e 'no-new-privileges' -e '--network' -e 'read-only'` in `sweagent config` → 5 hits (Hits are --memory=10g in three benchmark configs and the non-default `sweagent shell` host-repo mount (run_shell.py:129); the default config sets no container hardening.) (inferred)
  - *To reach the next level:* No hardening: non-root user, dropped capabilities, no-new-privileges, seccomp, read-only rootfs, or network-off by default.
- **C L3:** All actions go through SWEEnv.communicate to the SWE-ReX runtime, and tool bundles are uploaded and installed inside the container; host execution requires opting into another deployment type. — [sweagent/environment/swe_env.py:220-222](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/environment/swe_env.py#L220-L222); [sweagent/agent/agents.py:946-967](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L946-L967); [sweagent/tools/tools.py:297-309](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/tools/tools.py#L297-L309) (verified)
  - *To reach the next level:* Fail-closed behaviour and confinement of all spawned processes are not demonstrated in this repo's code (deployment start/fallback lives in SWE-ReX).
- **D L2:** Docker is the default deployment; switching to a local (host) deployment is an ordinary config option with no warning or per-call escalation. — [sweagent/environment/swe_env.py:27-30](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/environment/swe_env.py#L27-L30); [sweagent/run/run_single.py:19-26](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/run/run_single.py#L19-L26) (verified)
  - *To reach the next level:* Disabling the sandbox is not a loudly named operator flag with a warning.
- **B L0:** Inside the sandbox the model has root and unrestricted network egress, and GitHub token handling is not locked down. — [sweagent/environment/swe_env.py:27-30](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/environment/swe_env.py#L27-L30) (verified)
  - *To reach the next level:* Credentials must be kept out of the sandbox and egress restricted or allowlisted.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

SWE-agent's main input is a GitHub issue that anyone can write on a public repository, plus repository files and command output, all fed to the model with no provenance tracking or capability restriction. Issue text is wrapped in a <pr_description> tag but nothing acts on that. A hijacked session can make unrestricted network requests from the container with no human in the loop, and GitHub token handling widens the impact.

- **S L0:** No structural limit on a hijacked agent; untrusted issue text is templated directly into the task prompt. — [sweagent/utils/github.py:103-107](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/utils/github.py#L103-L107); [config/default.yaml:14-16](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/config/default.yaml#L14-L16) (verified)
  - *To reach the next level:* No rule-of-two enforcement or approval once untrusted content is read.
- **C L0:** Tool output enters history as ordinary user/tool messages with no untrusted marking. — [sweagent/agent/agents.py:702-710](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L702-L710) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished from the principal's instructions.
- **D L0:** No untrusted-input control exists to be on by default. — [sweagent/agent/agents.py:702-710](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L702-L710); searched `rg -n -i 'approv|confirm'` in `sweagent` → 0 hits (No approval or confirmation logic anywhere in the agent package.) (verified)
  - *To reach the next level:* No control to enable.
- **B L0:** A hijack can exfiltrate repository contents over the container's open network and take irreversible actions, unattended. — [sweagent/utils/github.py:103-107](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/utils/github.py#L103-L107); [sweagent/environment/swe_env.py:27-30](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/environment/swe_env.py#L27-L30) (verified)
  - *To reach the next level:* Either egress or irreversible state change would need human approval.
- **Cap:** C5-WORSTCASE — B is L0 in the default configuration: unattended exfiltration plus irreversible actions.

### C6 Memory, context & configuration integrity — 0.57 (high)

SWE-agent has no long-term memory, vector store, or auto-loaded instruction files, and each run starts in a fresh container, so a poisoned session cannot persist into the next one through the agent itself. Trajectories are saved on the host but are not read back into context. The one auto-loaded file is a .env from the current working directory, loaded silently, which can set the model endpoint and the GitHub token; it is not inside the agent's workspace by default, but it is loaded without any trust decision.

- **S L1:** A .env file in the current directory is loaded silently at startup and can redirect model endpoints or supply credentials. — [sweagent/utils/config.py:65-78](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/utils/config.py#L65-L78); [sweagent/run/run_single.py:166](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/run/run_single.py#L166) (verified)
  - *To reach the next level:* No trust decision before loading security-relevant settings from the working directory.
- **C L2:** There is no memory store or instruction-file loader to control; the cwd .env path is the one uncontrolled auto-load. — searched `rg -n -i 'AGENTS\.md|CLAUDE\.md|cursorrules|vector|remember'` in `sweagent` → 1 hits (Single hit is a docstring in tools/parsing.py ('Remember that all of the window's contents...'), not a memory or instruction-file loader.); [sweagent/utils/config.py:65-78](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/utils/config.py#L65-L78) (verified)
  - *To reach the next level:* The auto-loaded .env path is not covered by any control.
- **D SA:** No memory store exists, so there is no cross-user or cross-session namespace to isolate. — searched `rg -n -i 'AGENTS\.md|CLAUDE\.md|cursorrules|vector|remember'` in `sweagent` → 1 hits (Single hit is a docstring in tools/parsing.py ('Remember that all of the window's contents...'), not a memory or instruction-file loader.) (verified)
- **B L3:** Context is session-scoped: the container is ephemeral and trajectories are only written, never re-injected; persistence would require an opt-in host patch apply or an operator-placed .env. — [sweagent/environment/swe_env.py:27-30](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/environment/swe_env.py#L27-L30); [sweagent/agent/agents.py:1284-1286](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L1284-L1286); [sweagent/run/run_single.py:79](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/run/run_single.py#L79) (verified)
  - *To reach the next level:* Persistence paths (opt-in local patch apply, cwd .env) are not gated by human review with rollback.
- **Cap:** none

### C7 Third-party extensions — 0.05 (high)

Tool bundles are local directories chosen in the operator's config and uploaded into the container, but the default editor bundle installs an unpinned 'tree-sitter-languages' package from PyPI at every run, and the model can pip- or npm-install anything it likes via bash. Nothing verifies or pins what gets installed. Whatever runs is confined to the container rather than the host, but it runs as root next to the agent's working copy.

- **S L0:** The model can install arbitrary packages through the unrestricted shell, and the default bundle installs an unpinned PyPI package each run. — [tools/edit_anthropic/install.sh:3](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/tools/edit_anthropic/install.sh#L3); [sweagent/tools/tools.py:37-49](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/tools/tools.py#L37-L49) (verified)
  - *To reach the next level:* No pinning or integrity checks on packages installed at runtime.
- **C L0:** No extension type (bundles, their install scripts, model-chosen packages) is verified. — [sweagent/tools/tools.py:297-309](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/tools/tools.py#L297-L309); [tools/edit_anthropic/install.sh:3](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/tools/edit_anthropic/install.sh#L3) (verified)
  - *To reach the next level:* No verification on any extension type.
- **D L0:** Package installs happen automatically by default (bundle install.sh on startup, model installs at will). — [sweagent/tools/tools.py:297-309](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/tools/tools.py#L297-L309); [config/default.yaml:41-44](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/config/default.yaml#L41-L44) (verified)
  - *To reach the next level:* Third-party installs are not off by default or shown to the user for consent.
- **B L1:** Installed code runs as root inside the run's container with network access, but not on the host. — [sweagent/environment/swe_env.py:27-30](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/environment/swe_env.py#L27-L30) (verified)
  - *To reach the next level:* Extensions are not isolated per extension with scrubbed credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.15 (high)

The model API key is held as a masked SecretStr on the host and is not propagated into the container, and the repo contains no telemetry. Everything else is unprotected: per-instance trace, debug and info logs record every model input and command output unredacted, trajectories store full observations, and the opt-in PR feature posts the trajectory into the public PR body. The GitHub token, when set, is a long-lived credential whose handling is not locked down.

- **S L1:** Secrets come from env vars; only the model api_key is type-masked (SecretStr); no log or transcript redaction exists. — [sweagent/agent/models.py:85](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/models.py#L85); searched `rg -n -i 'redact|scrub|sanitiz'` in `sweagent` → 0 hits (No redaction helpers for logs, trajectories, or model-bound messages.) (verified)
  - *To reach the next level:* No log filters or redaction on main paths, and stored credentials are not protected from the model.
- **C L1:** Masking covers only the config dump of the model key; logs, trajectories and model-bound messages are unprotected. — [sweagent/agent/models.py:85](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/models.py#L85); [sweagent/agent/agents.py:701](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L701); [sweagent/tools/tools.py:89-91](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/tools/tools.py#L89-L91) (verified)
  - *To reach the next level:* Logs and transcripts are not redacted.
- **D L0:** No telemetry, but trace-level file logs of every model input and command output are written by default and unredacted (verbose payload logging on by default). — searched `rg -n -i 'telemetry|sentry|posthog'` in `sweagent pyproject.toml` → 0 hits (No telemetry SDK in the agent or its declared dependencies.); [sweagent/run/run_single.py:141-147](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/run/run_single.py#L141-L147) (verified)
  - *To reach the next level:* Verbose payload logs should be off by default or redacted.
- **B L0:** A long-lived user PAT (repo scope per .env.example) is used, and its handling is not locked down. — [.env.example:5-8](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/.env.example#L5-L8); [sweagent/run/hooks/open_pr.py:98](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/run/hooks/open_pr.py#L98) (verified)
  - *To reach the next level:* Credentials not scoped, short-lived, or kept out of model reach.
- **Cap:** none

### C9 Audit & traceability — 0.68 (high)

SWE-agent writes a structured trajectory (each action, observation, model response, execution time and state) after every step, plus timestamped trace/debug/info logs, all on the host outside the container the model controls. Blocked actions are recorded as well, and trajectories can be replayed. There is no tamper-evident storage, actor attribution beyond the agent name, or fail-closed behaviour if a write fails.

- **S L2:** Each step is recorded as structured JSON (action, observation, response, execution_time, state) with timestamps in the accompanying logs. — [sweagent/agent/agents.py:1220-1233](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L1220-L1233); [sweagent/run/run_single.py:141-147](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/run/run_single.py#L141-L147) (verified)
  - *To reach the next level:* No actor attribution separating human from agent or correlation IDs across runs.
- **C L3:** All tool calls (bash and bundles share one path), blocked actions, and retry attempts are recorded. — [sweagent/agent/agents.py:1088-1092](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L1088-L1092); [sweagent/agent/agents.py:1284-1286](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L1284-L1286) (verified)
  - *To reach the next level:* Configuration changes and credential use (PR push) are not in the trajectory.
- **D L3:** On by default and written by the host Python process to cwd/trajectories, outside the container where model commands run. — [sweagent/run/run_single.py:79](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/run/run_single.py#L79); [sweagent/agent/agents.py:1284-1286](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L1284-L1286); [sweagent/environment/swe_env.py:220-222](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/environment/swe_env.py#L220-L222) (verified)
  - *To reach the next level:* Logging can be disabled or redirected by config without a logged operator change.
- **B L3:** The trajectory is rewritten to disk after every step, so a crash loses at most one step and the run can be replayed. — [sweagent/agent/agents.py:1284-1286](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L1284-L1286) (verified)
  - *To reach the next level:* Actions are not blocked when their record cannot be written (no fail-closed).
- **Cap:** none

### C10 Limits & kill switch — 0.42 (medium)

Each run is capped at $3 of model spend by default, and the agent refuses to run with that cap if it cannot price the model, rather than silently running unbounded. Each command times out after 30 seconds and is interrupted, total command time is capped at 30 minutes (checked between steps), and retry attempts share one budget. There is no step cap or wall-clock limit, background processes keep running until the container is torn down, and the single-run CLI has no finally block guaranteeing container teardown on errors.

- **S L1:** Per-instance cost cap ($3, fail closed on unknown pricing), per-command 30s timeout with interrupt, and a cooperative 1800s total-execution cap, but no iteration cap. — [sweagent/agent/models.py:73-78](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/models.py#L73-L78); [sweagent/agent/models.py:660-665](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/models.py#L660-L665); [sweagent/agent/models.py:743-754](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/models.py#L743-L754); [sweagent/tools/tools.py:139-148](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/tools/tools.py#L139-L148); [sweagent/agent/agents.py:968-981](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L968-L981); searched `rg -n -i 'max_steps|max_iterations|step_limit'` in `sweagent` → 0 hits (No step/iteration cap exists.); searched `rg -n -i 'wall|deadline'` in `sweagent` → 0 hits (No wall-clock limit exists.) (verified)
  - *To reach the next level:* No step/iteration cap (the L2 anchor requires one) and no wall-clock limit.
- **C L2:** Limits apply to the top-level loop and to each command; retry attempts share the cost budget. — [sweagent/agent/agents.py:1018-1019](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L1018-L1019); [sweagent/agent/agents.py:968-981](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L968-L981); [sweagent/agent/agents.py:307-310](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/agents.py#L307-L310) (verified)
  - *To reach the next level:* Background processes spawned inside the container are not counted against any budget.
- **D L2:** Sensible defaults set in host-side config the model cannot reach; there is no delegation to reset them (capped at one level above Strength). — [sweagent/agent/models.py:73-78](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/models.py#L73-L78); [sweagent/tools/tools.py:139-148](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/tools/tools.py#L139-L148); [sweagent/agent/models.py:660-665](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/agent/models.py#L660-L665) (verified)
  - *To reach the next level:* Strength must reach L2 (add a step cap) before the model-cannot-raise property earns L3; limits can also be set to 0 = unlimited.
- **B L2:** Ceilings are moderate ($3, 30 min command time); stopping ends the loop and the container is killed on normal close, but background work runs until then and run_single lacks a finally for teardown. — [sweagent/run/run_single.py:188-207](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/run/run_single.py#L188-L207); [sweagent/run/run_batch.py:370-371](https://github.com/SWE-agent/SWE-agent/blob/3ea751c087f32b16e039a2233dd6eefecef325d5/sweagent/run/run_batch.py#L370-L371) (inferred)
  - *To reach the next level:* Teardown on stop/error is not guaranteed in single-run mode and no provider-side spend cap exists.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: GitHub issue title/body from anyone (sweagent/utils/github.py:103-107) and repository/command output (sweagent/agent/agents.py:702-710) · [B] sensitive data/systems: GITHUB_TOKEN; private repo contents when the token grants them · [C] state change / egress: Unrestricted shell and network inside the container (sweagent/environment/swe_env.py:27-30, 220-222) · Same default session? Yes

## Highest-impact improvements
1. Keep GitHub credentials out of the model-controlled environment and push only from the host after review. — C1 B L0→L2, +0.100 before caps (Playbook 4)
2. Default the container to no network (or an allowlisted proxy) after setup. — C4 B L0→L2, +0.100 before caps (Playbook 3)
3. Harden the default container: pass non-root user, --cap-drop=ALL, --security-opt=no-new-privileges, and memory/PID limits via docker_args in the default EnvironmentConfig. — C4 S L2→L3, +0.075 before caps (Playbook 3)
4. Redact secret patterns (tokens, keys) from trace/debug logs, trajectories, and the PR-body trajectory dump. — C8 S L1→L2, +0.075 before caps (Playbook 4)
5. Add a default step cap and wall-clock limit, and wrap single-run execution in try/finally so the container is always torn down. — C10 S L1→L2, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Container behaviour (docker run arguments, no mounts, --rm, network) lives in the SWE-ReX dependency (swe-rex>=1.4.0, unpinned); it was read at SWE-ReX main 5c995c3 for context but is not citable at the pinned commit, so C4 Strength is marked inferred.
- Scoring assumes GITHUB_TOKEN is set, as .env.example and docs/installation/keys.md ask; without a token C1, C4, C5 and C8 blast-radius ratings would improve, though egress and the lack of approval remain.
- Non-default modes not scored: `sweagent shell` mounts the host repository read-write into the container (sweagent/run/run_shell.py:129-132); --env.deployment.type=local runs on the host; modal runs remotely; open_pr and apply_patch_locally act outside the container.
- litellm (model client) behaviour, including any callbacks or telemetry it may enable, was not examined.
- No reviewer-injection attempts were found in README, docs, .cursor rules, or tool prompts.
