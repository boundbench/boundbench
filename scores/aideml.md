# Defense-in-Depth Score: AIDE ML

**Repo:** https://github.com/wecoai/aideml · **Commit:** `60b3978ddf65b71f86eb7c64506965048a1398cf` · **Reviewed:** 2026-10-04
**What it is:** Tree-search ML-engineering agent that writes, runs and iteratively improves Python training scripts against a dataset and metric.
**Category:** Data & Analytics
**Scored configuration:** The `aide` CLI after `pip install aideml`, packaged config.yaml defaults (20 steps, 1-hour exec timeout), run on the host as in the README quick start.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 2.9 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | C2-POWERBYPASS | **0.00** | High |
| C3 | Tool & action scoping | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C4 | Code-execution isolation | L2 | L3 | L0 | L1 | 0.42 | G1 | **0.42** (alt) | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | Medium |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |

Controls where a risk surface exists: 1.88 / 9.0 (21%); 1 criterion scored SA (surface absent).

As shipped, AIDE runs every Python script the model writes directly on your machine, as you, with your environment (including your LLM API keys) and open network access, and no human review. Dataset files are pasted into the model's system prompt, so a poisoned dataset can steer code that steals credentials or deletes files. Its only safeguards are a step count and a per-script timeout; the shipped Dockerfile is the main mitigation and is opt-in.

## Critical gaps
- Arbitrary model-written Python runs with no approval gate in the default configuration (C2-POWERBYPASS). (ASI02, ASI09, T2; C2) — [aide/agent.py:391-394](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/agent.py#L391-L394)
- Generated code runs via exec() in a same-user child process with full builtins, inherited environment and network: no isolation. (ASI05, T11, LLM05; C4) — [aide/interpreter.py:143-156](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L143-L156); [aide/interpreter.py:185-189](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L185-L189)
- Dataset content enters the system prompt and the resulting code runs unattended with credentials and egress (C5-WORSTCASE). (ASI01, LLM01, T6; C5) — [aide/utils/data_preview.py:137-143](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/utils/data_preview.py#L137-L143); [aide/agent.py:302-303](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/agent.py#L302-L303)
- Executed code inherits the operator's full ambient authority and LLM API keys. (ASI03, T3; C1) — [aide/interpreter.py:185-189](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L185-L189); [aide/backend/backend_openai.py:32](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/backend/backend_openai.py#L32)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

AIDE has no agent identity or authorization layer. The LLM-written training script runs in a child process of the agent, as the user who launched it, with the full environment inherited, so the model's code can read the LLM API keys and anything else the user can reach (home directory, cloud CLI credentials, SSH keys). Nothing narrows that ambient authority.

- **S L0:** Generated code runs with the operator's full ambient authority, including LLM API keys read from the environment. — [aide/interpreter.py:185-189](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L185-L189); [aide/backend/backend_openai.py:32](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/backend/backend_openai.py#L32) (verified)
  - *To reach the next level:* No dedicated or narrowed identity for executed code; L1 needs at least a separate identity.
- **C L0:** No authorization check exists on the execution path and the child process inherits os.environ unchanged. — [aide/agent.py:391-394](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/agent.py#L391-L394); searched `rg -n 'env='` in `aide/` → 0 hits (no subprocess/Process call passes a restricted environment) (verified)
  - *To reach the next level:* No authorization layer; L1 needs the main tool path checked in code.
- **D L0:** The default install runs generated code with the launching user's privileges; there is no narrower mode. — [aide/interpreter.py:143-156](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L143-L156) (verified)
  - *To reach the next level:* No narrower default; L1 needs any default narrower than the user's authority.
- **B L0:** A hijacked script can use everything the OS user can: all files, cloud/SSH credentials, and the LLM provider keys. — [aide/interpreter.py:143-156](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L143-L156); [aide/backend/backend_openai.py:32](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/backend/backend_openai.py#L32) (verified)
  - *To reach the next level:* Nothing scopes reachable credentials; L1 needs reach limited below the user's entire account.
- **Cap:** none
- **Notes:** The Dockerfile runs as a non-root user inside a container, which narrows host reach; that is credited under C4 as an opt-in mode.

### C2 Approval gates — 0.00 (high)

There is no human approval anywhere. Every step, the model writes a complete Python program and AIDE executes it immediately, for 20 steps by default, with no review of the code. The single most powerful action, arbitrary code execution, is the only action and it is never gated.

- **S L0:** No approval mechanism exists; code is executed as soon as it is generated. — [aide/agent.py:391-394](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/agent.py#L391-L394); searched `rg -n -i 'approv|confirm|input\('` in `aide/` → 3 hits (all 3 hits are Streamlit text_input fields for API keys in the web UI; none gates execution) (verified)
  - *To reach the next level:* No approval; L1 needs at least a blanket human approval before execution.
- **C L0:** The code-execution path, the most powerful action, is exempt because no gate exists. — [aide/agent.py:391-394](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/agent.py#L391-L394); [aide/run.py:135-136](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/run.py#L135-L136) (verified)
  - *To reach the next level:* No gate on the exec path; L1 needs at least flagged tools gated.
- **D L0:** There is no approval to turn on; the default run loop is fully autonomous. — [aide/run.py:135-136](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/run.py#L135-L136) (verified)
  - *To reach the next level:* Approval is not available at all; L1 needs it on by default.
- **B L0:** Executed code can delete files, send data anywhere, or install software, with no undo or checkpoint. — [aide/interpreter.py:143-156](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L143-L156) (verified)
  - *To reach the next level:* No rollback or preview; L1 needs some reversible actions.
- **Cap:** C2-POWERBYPASS — Arbitrary Python execution, the agent's most powerful path, runs without any gate in the default configuration.

### C3 Tool & action scoping — 0.00 (high)

AIDE's only action is 'run this whole Python program'. There is no argument validation, path containment, network allowlist or quantity bound; the prompt even tells the model it may use any package. The workspace directory is only the working directory, not a boundary.

- **S L0:** Raw passthrough: the model's entire program is compiled and executed with full builtins. — [aide/interpreter.py:143-156](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L143-L156) (verified)
  - *To reach the next level:* No validation; L1 needs at least denylist filtering.
- **C L0:** The single execution path validates nothing. — [aide/agent.py:391-394](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/agent.py#L391-L394) (verified)
  - *To reach the next level:* No tool validates input; L1 needs some validation on some tool.
- **D L0:** Code execution with filesystem, network and process access is the default and cannot be narrowed by config. — [aide/interpreter.py:143-156](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L143-L156); [aide/agent.py:214](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/agent.py#L214) (verified)
  - *To reach the next level:* Everything enabled; L1 needs dangerous capabilities individually disableable.
- **B L0:** A misused 'tool' is any program on the whole machine; os.chdir to the workspace is not containment. — [aide/interpreter.py:123](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L123); [aide/interpreter.py:143-156](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L143-L156) (verified)
  - *To reach the next level:* Reach is the whole machine; L1 needs some limit on reach.
- **Cap:** none

### C4 Code-execution isolation — 0.42 (medium)

By default, model-written code is executed with Python's exec() inside a forked child process on the host, as the same user, with the full environment (including LLM API keys) and unrestricted network. The only containment is a wall-clock timeout. The repository also ships a Dockerfile that runs AIDE as a non-root user in a stock container; that narrows host reach, but it is an optional deployment mode, the container still receives the API key, and it has no hardening.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Same-user subprocess running in-process exec() with full __builtins__; no isolation primitive. — [aide/interpreter.py:143-156](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L143-L156); searched `rg -n -i 'sandbox|docker|seccomp|setrlimit|resource\.'` in `aide/` → 0 hits (no isolation or resource-limit code anywhere in the package) (verified)
    - *To reach the next level:* No isolation; L1 needs at least filtering or a separate restricted runtime.
  - **C L0:** The main and only exec path is unsandboxed. — [aide/agent.py:391-394](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/agent.py#L391-L394); [aide/interpreter.py:185-189](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L185-L189) (verified)
    - *To reach the next level:* Main exec path not sandboxed; L1 needs the main exec tool sandboxed.
  - **D L0:** No sandbox is on by default in the CLI or Python API. — [aide/run.py:82-85](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/run.py#L82-L85) (verified)
    - *To reach the next level:* No default isolation; L1 needs isolation on by default.
  - **B L0:** Executed code reaches the user's home directory, all credentials in the inherited environment, and the network. — [aide/interpreter.py:143-156](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L143-L156); [aide/backend/backend_openai.py:32](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/backend/backend_openai.py#L32) (verified)
    - *To reach the next level:* Host-equivalent reach; L1 needs at least credentials out of the execution environment or reduced mounts.
- **shipped Dockerfile (non-root stock container)** (alt; raw 0.42, cap G1 → 0.42) ← counted
  - **S L2:** Stock container with a dedicated non-root user; no capability drops, seccomp profile or read-only root. — [Dockerfile:36](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/Dockerfile#L36); [Dockerfile:52](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/Dockerfile#L52) (verified)
    - *To reach the next level:* No hardening (cap drop, no-new-privileges, seccomp, read-only rootfs); L3 needs a hardened container.
  - **C L3:** In container mode both the agent and every script it executes run inside the container (single ENTRYPOINT). — [Dockerfile:55](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/Dockerfile#L55); [Dockerfile:52](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/Dockerfile#L52) (verified)
    - *To reach the next level:* No fail-closed guarantee for processes spawned outside the container; L4 needs every path including spawned processes confined with no fallback.
  - **D L0:** Container mode is opt-in; the README leads with pip install and running on the host. — [README.md:72-75](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/README.md#L72-L75) (verified)
    - *To reach the next level:* Off by default; L1 needs isolation on by default.
  - **B L1:** The documented run mounts logs/workspaces/data read-write, passes OPENAI_API_KEY into the container, and leaves network egress open. — [README.md:182-186](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/README.md#L182-L186) (inferred)
    - *To reach the next level:* Credentials are in the sandbox environment with full egress; L2 needs credentials out of the sandbox.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.00 (high)

Dataset files in the user's data directory are previewed into the model's system prompt, and small text files are pasted in verbatim, alongside the code's own execution output. Nothing distinguishes this content from instructions, and the model's next action is a program that runs with network access and the user's credentials. A poisoned dataset file can therefore lead to credential theft and destructive actions with no human involved.

- **S L0:** No structural limit: untrusted data and instructions share one system message and the output is executed directly. — [aide/agent.py:302-303](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/agent.py#L302-L303); [aide/agent.py:258-260](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/agent.py#L258-L260) (verified)
  - *To reach the next level:* No defense at all; L1 needs at least delimiting or detection of untrusted content.
- **C L0:** Dataset content and execution output enter the system prompt with the same standing as the task description. — [aide/utils/data_preview.py:137-143](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/utils/data_preview.py#L137-L143); [aide/agent.py:410](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/agent.py#L410) (verified)
  - *To reach the next level:* Untrusted sources not distinguished; L1 needs one source handled.
- **D L0:** No control exists to enable. — [aide/utils/data_preview.py:137-143](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/utils/data_preview.py#L137-L143) (verified)
  - *To reach the next level:* Nothing on by default; L1 needs a control on by default.
- **B L0:** A hijacked run can exfiltrate keys and data over the network and delete or modify files, unattended. — [aide/interpreter.py:143-156](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L143-L156); [aide/backend/backend_openai.py:32](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/backend/backend_openai.py#L32) (verified)
  - *To reach the next level:* Both leak and irreversible action are unattended; L1 needs at least one of them to require a human.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.45 (medium)

AIDE has no long-term memory: the in-run 'Memory' summary lives only in the current process and the saved journal is never read back. Configuration comes from the package's own config.yaml and command-line arguments, not from the data or workspace directory. However, nothing protects that configuration or the Python environment from the agent's own generated code, which runs as the same user and could plant files that persist into future runs.

- **S L2:** No memory store; settings load only from the package config and CLI, and dataset files are read as data, not configuration. — [aide/utils/config.py:96-102](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/utils/config.py#L96-L102); searched `rg -n 'load_json'` in `aide/` → 1 hits (only the definition in serialize.py; the journal is never loaded back) (verified)
  - *To reach the next level:* No protection of config files from the agent's own code; L3 needs gated writes and an explicit trust step for security-relevant config.
- **C L2:** The config path is user scope, and the run journal is write-only; but the Python environment and config file are writable by executed code. — [aide/utils/config.py:187-199](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/utils/config.py#L187-L199) (verified)
  - *To reach the next level:* Auto-loaded package files are not controlled against agent writes; L3 needs all auto-loaded files controlled.
- **D L2:** Each run gets its own numbered workspace and log directory. — [aide/utils/config.py:137-138](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/utils/config.py#L137-L138) (verified)
  - *To reach the next level:* Executed code can write other runs' directories; L3 needs the model unable to write other namespaces.
- **B L1:** Because generated code runs as the user, a poisoned run can persist code (e.g. into the Python environment) that executes in later sessions. — [aide/interpreter.py:143-156](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L143-L156) (inferred)
  - *To reach the next level:* Persistence can trigger code execution in later sessions; L2 needs persisted content limited to text output.
- **Cap:** none
- **Notes:** The Streamlit web UI calls load_dotenv() (aide/webui/app.py:50); python-dotenv's default search starts from the calling file's directory, i.e. the package checkout, not a dataset directory. The web UI is a secondary mode.

### C7 Third-party extensions — 1.00 (high)

AIDE loads no plugins, MCP servers, hub tools or remote prompts. Generated code can import or install anything, but that is arbitrary code execution and is scored under code-execution isolation, not as an extension system.

- **Structural absence:** searched `rg -n -i 'plugin|mcp|entry_points|importlib|trust_remote_code|torch\.load|pickle'` in `aide/` → 1 hits (single hit is a traceback filter string in interpreter.py:55, not a loader)

### C8 Secrets & sensitive-data protection — 0.05 (high)

API keys are read from environment variables and handed, along with the rest of the environment, to the process that runs model-written code. There is no masking or redaction anywhere; execution output (which a script could fill with environment variables) is sent back to the model and saved in the run journal. There is no telemetry, and verbose request logging is off by default.

- **S L0:** Keys come from env vars with no masking, redaction, or secret typing on any path. — [aide/backend/backend_openai.py:32](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/backend/backend_openai.py#L32); searched `rg -n -i 'redact|mask|scrub|SecretStr'` in `aide/` → 0 hits (verified)
  - *To reach the next level:* No masking on any path; L1 needs masking on at least one path.
- **C L0:** No path is protected: subprocess env is inherited, execution output goes to the model and journal unfiltered. — searched `rg -n 'env='` in `aide/` → 0 hits (no subprocess/Process call passes a restricted environment); [aide/agent.py:410](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/agent.py#L410) (verified)
  - *To reach the next level:* No path protected; L1 needs one path protected.
- **D L1:** No telemetry exists and logging defaults to WARNING so full prompts are not logged, but there is no redaction to keep on. — [aide/utils/config.py:19-23](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/utils/config.py#L19-L23); searched `rg -n -i 'sentry|posthog|telemetry|segment|wandb'` in `aide/` → 0 hits (verified)
  - *To reach the next level:* No redaction exists; L2 needs a redaction layer on by default.
- **B L0:** Long-lived provider keys, plus whatever the user's home directory holds, are reachable by every executed script. — [aide/interpreter.py:185-189](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L185-L189); [aide/backend/backend_openai.py:32](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/backend/backend_openai.py#L32) (verified)
  - *To reach the next level:* Long-lived keys reachable by executed code; L1 needs keys kept from executed code or scoped.
- **Cap:** none
- **Notes:** The web UI writes keys pasted into the sidebar into os.environ (aide/webui/app.py:353), so they also reach executed code in that mode.

### C9 Audit & traceability — 0.50 (high)

After every step AIDE saves a structured journal (each script, its output, execution time, the reviewer's analysis and a timestamp) plus the best solution into a per-run log directory. That is a usable record of what ran. It has no actor attribution or integrity protection, it is written after execution, and the executed code runs as the same user so it can alter or delete the log.

- **S L2:** Structured per-node record of code, output, exec time and creation timestamp. — [aide/journal.py:35](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/journal.py#L35); [aide/journal.py:40-41](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/journal.py#L40-L41) (verified)
  - *To reach the next level:* No actor/approver attribution or correlation IDs; L3 needs actor attribution.
- **C L2:** Every executed script is recorded; LLM requests are only logged at INFO, which is off by default. — [aide/utils/config.py:187-199](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/utils/config.py#L187-L199); [aide/backend/backend_openai.py:95](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/backend/backend_openai.py#L95) (verified)
  - *To reach the next level:* No approvals exist to record and LLM calls aren't in the default record; L3 needs all calls plus approvals/denials.
- **D L2:** On by default and stored in logs/<run>, outside the workspace, but writable by the same user the generated code runs as. — [aide/utils/config.py:137](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/utils/config.py#L137) (verified)
  - *To reach the next level:* Executed code can alter the record; L3 needs a writer the model can't control.
- **B L2:** The journal is rewritten after each step so records are flushed per action; failures raise. — [aide/run.py:136-137](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/run.py#L136-L137) (verified)
  - *To reach the next level:* Overwritten in place, not append-only or replay-durable; L3 needs durable per-action records.
- **Cap:** none

### C10 Limits & kill switch — 0.45 (high)

Runs are bounded by a step count (20 by default) and a per-script timeout (one hour by default) that interrupts and then kills the child process. There is no overall time budget and no token or cost cap, and the timeout kills only the direct child, so processes a script spawns itself can survive. Ctrl+C stops the run.

- **S L2:** Step cap plus a per-execution timeout enforced with SIGINT then terminate/kill. — [aide/utils/config.yaml:36](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/utils/config.yaml#L36); [aide/interpreter.py:280-288](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L280-L288) (verified)
  - *To reach the next level:* No wall-clock or token/cost cap; L3 needs all three plus rate limits.
- **C L2:** Limits cover the top-level loop and each execution, but cleanup signals only the child PID, not its process group. — [aide/interpreter.py:196-206](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/interpreter.py#L196-L206); [aide/run.py:135](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/run.py#L135) (verified)
  - *To reach the next level:* Processes spawned by generated code are not counted or killed; L3 needs spawned processes under the same budget.
- **D L2:** Defaults are set in the packaged config and changed only by the operator via CLI; the model can't change them. — [aide/utils/config.yaml:23](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/utils/config.yaml#L23) (verified)
  - *To reach the next level:* Generated code runs as the user and could edit the packaged config.yaml to raise limits for later runs; L3 needs limits the model cannot raise.
- **B L1:** Ceilings are large: 20 steps of up to an hour each, with no spend cap, and grandchild processes may outlive a stop. — [aide/utils/config.yaml:23](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/utils/config.yaml#L23); [aide/backend/utils.py:13](https://github.com/wecoai/aideml/blob/60b3978ddf65b71f86eb7c64506965048a1398cf/aide/backend/utils.py#L13) (verified)
  - *To reach the next level:* No tight per-run time or cost ceiling; L2 needs moderate ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: dataset files previewed (small text files verbatim) into the system prompt, aide/utils/data_preview.py:137-143, aide/agent.py:302-303 · [B] sensitive data/systems: LLM API keys and the user's home directory via inherited environment, aide/backend/backend_openai.py:32, aide/interpreter.py:185-189 · [C] state change / egress: model-written code exec()'d with full builtins and network, aide/interpreter.py:156 · Same default session? Yes

## Highest-impact improvements
1. Start the interpreter child with a scrubbed environment (no *_API_KEY variables) so executed scripts can't read provider keys. — C8 C L0→L1, +0.075 before caps (Playbook 4)
2. Run each generated script in a hardened, network-off container (non-root, cap-drop, read-only root, workspace-only mount) by default. — C4 S L0→L3, +0.225 before caps (Playbook 3)
3. Make container execution the default for the CLI and Python API, with host execution behind an explicit flag. — C4 D L0→L3, +0.150 before caps (Playbook 3)
4. Add a review step that shows the exact script before it runs, on by default for interactive CLI use. — C2 S L0→L3, +0.225 before caps (Playbook 5)
5. Add a session wall-clock and token/cost budget, and kill the child's whole process group on timeout or stop. — C10 S L2→L3, +0.075 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The Streamlit web UI and Python API were reviewed only where they share the CLI's code paths; the web UI's network exposure was not examined.
- Docker run flags come from the README, not code; the container B rating is inferred.
- Child-process environment inheritance relies on Python multiprocessing behaviour (fork/forkserver/spawn all inherit os.environ); no start method is set in the code.
- Minor: the generated run-visualisation output is not locked down; this is not scored separately.
- No text aimed at AI reviewers was found (searched for reviewer/auditor/ignore-previous phrasing).
