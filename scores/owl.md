# Defense-in-depth score: OWL

**Repo:** https://github.com/camel-ai/owl · **Commit:** `d67601b245bb91a38876c603cb8a70aa2f5faae8` · **Reviewed:** 2026-10-05
**What it is:** CAMEL-AI's multi-agent workforce for general task automation: web research, browser use, document processing and code execution.
**Category:** AI Assistants
**Scored configuration:** README Quick Start: python examples/run.py with an OpenAI key in owl/.env, on the host, CAMEL 0.2.84 defaults, no edits.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents yes · external communication yes

## Score: 2.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | Medium |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C3 | Tool & action scoping | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C4 | Code-execution isolation | L2 | L3 | L0 | L0 | 0.38 | G1 | **0.38** (alt) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | none | **0.10** | Medium |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | Medium |
| C9 | Audit & traceability | L1 | L2 | L1 | L0 | 0.28 | none | **0.28** | Medium |
| C10 | Limits & kill switch | L1 | L2 | L1 | L1 | 0.33 | none | **0.33** | Low |

Controls where a risk surface exists: 1.07 / 9.0 (12%); 1 criterion scored SA (surface absent).

OWL runs a team of AI agents that browse the web, read documents and write and run code, and as shipped none of it is contained. Model-written Python and shell code runs directly on your machine as your user, with every API key from owl/.env in its environment, and there is no approval step, no sandbox and nothing limiting what injected web content can make the agents do. Run it only in a disposable VM or container that holds no valuable credentials or data.

## Critical gaps
- Agents run with the launching user's full authority, and model-written code inherits every API key and the user's home directory. (ASI03, T3; C1). Evidence: [examples/run.py:38-40](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L38-L40); [examples/run.py:87](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L87)
- Model-written Python and shell code runs on the host as the user, with no sandbox and no approval, by default. (ASI05, T11; C4). Evidence: [examples/run.py:87](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L87); [examples/run.py:140-144](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L140-L144)
- Web pages and documents the agents read can steer a workforce that runs host code, so a hijack can leak keys and files and take destructive actions unattended. (ASI01, LLM01; C5). Evidence: [examples/run.py:117-123](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L117-L123); [examples/run.py:128-134](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L128-L134); [owl/utils/document_toolkit.py:166](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/owl/utils/document_toolkit.py#L166)

## Criterion details

### C1 Identity & least privilege: 0.00 (medium confidence)

OWL has no identity or authorization layer. It loads every key in owl/.env into its own process environment at startup, and the agents act with the full authority of the user who launched them. Model-written code runs as that user on the host and, through the CAMEL library, inherits the whole environment, so every API key, cloud CLI login, SSH key and file the user can reach is reachable by the agents. Nothing narrows or checks this per request.

- **S L0:** All API keys from owl/.env are loaded into the process environment and the agents run with the launching user's ambient authority. Evidence: [examples/run.py:38-40](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L38-L40); [examples/run.py:87](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L87) (verified)
  - *To reach the next level:* No dedicated or scoped identity; keys and OS permissions are used as-is.
- **C L0:** camel-ai 0.2.84 behaviour, read at its v0.2.84 tag: the subprocess code interpreter passes a full copy of os.environ to every child, so model-written code receives every credential. Evidence: [examples/run.py:87](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L87); [examples/run.py:128-134](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L128-L134) (inferred)
  - *To reach the next level:* No tool path goes through an authorization check, and subprocesses are not given a scrubbed environment.
- **D L0:** The default run gives every worker the user's full privilege with no hardening option in the shipped configuration. Evidence: [examples/run.py:38-40](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L38-L40); [examples/run.py:140-144](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L140-L144) (verified)
  - *To reach the next level:* No narrower default role; least privilege requires the user to run OWL under a separate account or container.
- **B L0:** A hijacked agent can use everything the OS user can: home directory, SSH keys, cloud and Git CLIs, and all model and tool API keys. Evidence: [examples/run.py:87](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L87); [examples/run.py:220](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L220) (inferred)
  - *To reach the next level:* Blast radius is the user's whole account across services; nothing confines it to one project or read-only access.
- **Cap:** none

### C2 Approval gates: 0.00 (high confidence)

No action needs a human's approval. The example agents call CAMEL's code-execution toolkit without its confirmation option, so model-written Python and shell code runs immediately, and file writes, browser actions and web requests have no gate either. Mistakes and hijacked actions, including deleting files or running arbitrary commands, happen with no chance to stop them, and nothing offers undo.

- **S L0:** No approval mechanism is used; the code tool is constructed without a confirmation flag and the CAMEL toolkit defaults it to off. Evidence: [examples/run.py:87](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L87); searched `rg -n -S -e 'require_confirm|approv|confirm|human_in_the_loop|HumanToolkit'` in `owl examples` → 0 hits (no approval step or confirmation flag anywhere in OWL's own code or examples) (verified)
  - *To reach the next level:* No per-call human approval showing the exact code, file write or browser action.
- **C L0:** The most powerful tool, host code execution, is ungated in both workers that hold it. Evidence: [examples/run.py:128-134](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L128-L134); [examples/run.py:140-144](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L140-L144) (verified)
  - *To reach the next level:* Code execution, file writes and browsing would all need to cross a gate.
- **D L0:** Approval is not available in the shipped configuration at all. Evidence: searched `rg -n -S -e 'require_confirm|approv|confirm|human_in_the_loop|HumanToolkit'` in `owl examples` → 0 hits (no approval step or confirmation flag anywhere in OWL's own code or examples) (verified)
  - *To reach the next level:* Approval is not on by default.
- **B L0:** Ungated host code can delete or overwrite any user file and send data anywhere; there is no checkpoint or undo. Evidence: [examples/run.py:220](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L220); [examples/run.py:87](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L87) (verified)
  - *To reach the next level:* No checkpoints, rollback or previews for file and external actions.
- **Cap:** none

### C3 Tool & action scoping: 0.00 (high confidence)

The default tool set is as broad as it gets: arbitrary Python, shell and R code, file reads and writes anywhere on disk, arbitrary URL fetching and a full browser. OWL's own document tool opens any local path or URL with no containment, follows redirects and does not block internal addresses. No argument is validated against an allowlist and every tool is enabled by default.

- **S L0:** Tools are raw passthrough: arbitrary code, arbitrary local paths and arbitrary URLs with redirects followed. Evidence: [examples/run.py:87](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L87); [owl/utils/document_toolkit.py:94-104](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/owl/utils/document_toolkit.py#L94-L104); [owl/utils/document_toolkit.py:166](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/owl/utils/document_toolkit.py#L166); searched `rg -n -S -e 'allowed_|is_relative_to|realpath|169.254|localhost|allowlist|whitelist'` in `owl examples` → 4 hits (all hits are the default vLLM endpoint URL in run_vllm.py; no path, URL or host allowlist) (verified)
  - *To reach the next level:* No resolved-path containment, URL/host allowlist or internal-address block.
- **C L0:** No tool in the default set validates its inputs. Evidence: [owl/utils/document_toolkit.py:94-104](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/owl/utils/document_toolkit.py#L94-L104); [examples/run.py:88](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L88); searched `rg -n -S -e 'allowed_|is_relative_to|realpath|169.254|localhost|allowlist|whitelist'` in `owl examples` → 4 hits (all hits are the default vLLM endpoint URL in run_vllm.py; no path, URL or host allowlist) (verified)
  - *To reach the next level:* Even a few tools validating their inputs would be L1.
- **D L0:** Exec, write, network and browser tools are all enabled by default for the worker agents. Evidence: [examples/run.py:117-123](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L117-L123); [examples/run.py:128-134](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L128-L134); [examples/run.py:140-144](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L140-L144) (verified)
  - *To reach the next level:* Dangerous tools are not individually disableable in configuration; changing them means editing the script.
- **B L0:** A misused tool can run any command, touch any file and reach any host the machine can. Evidence: [examples/run.py:87](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L87); [owl/utils/document_toolkit.py:166](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/owl/utils/document_toolkit.py#L166) (verified)
  - *To reach the next level:* Tools are not scoped to a workspace or bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation: 0.38 (high confidence)

In the default setup, model-written Python, shell and R code runs directly on your machine as your user. OWL selects CAMEL's 'subprocess' backend, which is an ordinary child process with the full environment, including every API key, and no isolation. The project also ships a Docker setup that runs the whole app in a container; it is opt-in, runs as root in a stock image with full network, and mounts the .env file and host cache directories, so it contains host damage but not credential or data loss.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** The 'subprocess' backend runs code as a same-user host process with no isolation. Evidence: [examples/run.py:87](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L87); searched `rg -n -S -e 'docker|e2b|sandbox|seccomp|landlock'` in `owl examples` → 7 hits (all 7 hits select the 'subprocess' backend (host execution) in each example script) (verified)
    - *To reach the next level:* Not even filtering or a separate low-privilege user; a container or OS sandbox would be needed.
  - **C L0:** Every worker that runs code, including workers CAMEL creates at runtime, uses host execution. Evidence: [examples/run.py:128-134](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L128-L134); [examples/run.py:140-144](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L140-L144); [examples/run.py:200-204](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L200-L204) (verified)
    - *To reach the next level:* The main exec tool is not sandboxed.
  - **D L0:** No isolation is configured by default in any example script. Evidence: searched `rg -n -S -e 'docker|e2b|sandbox|seccomp|landlock'` in `owl examples` → 7 hits (all 7 hits select the 'subprocess' backend (host execution) in each example script) (verified)
    - *To reach the next level:* Isolation is not on by default.
  - **B L0:** camel-ai 0.2.84 behaviour, read at its v0.2.84 tag: the child process inherits the full environment, so code reaches all API keys, the user's home directory and the network. Evidence: [examples/run.py:38-40](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L38-L40); [examples/run.py:87](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L87) (inferred)
    - *To reach the next level:* Code would need to run without secrets, without home-directory access and with network limits.
- **opt-in Docker deployment** (alt; raw 0.38, cap G1 → 0.38) ← counted
  - **S L2:** The shipped image is a stock python:3.10-slim container with no USER directive, so code runs as root with default capabilities. Evidence: [.container/Dockerfile:1](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/.container/Dockerfile#L1); [.container/Dockerfile:58-59](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/.container/Dockerfile#L58-L59) (verified)
    - *To reach the next level:* No non-root user, dropped capabilities, seccomp profile or read-only root filesystem.
  - **C L3:** When deployed this way, the whole application and every exec path run inside the container. Evidence: [README.md:286-292](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/README.md#L286-L292) (verified)
    - *To reach the next level:* No fail-closed guarantee; the same scripts run on the host when started outside the container.
  - **D L0:** The Docker deployment is one of four install options; the Quick Start runs on the host. Evidence: [README.md:394](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/README.md#L394) (verified)
    - *To reach the next level:* Container execution is opt-in.
  - **B L0:** The container receives the API key in its environment and the .env file, and mounts host cache directories read-write, with full network. Evidence: [.container/docker-compose.yml:12-23](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/.container/docker-compose.yml#L12-L23) (verified)
    - *To reach the next level:* Credentials are present inside the container and host directories are mounted read-write.
- **Cap:** G1: Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius: 0.00 (high confidence)

OWL's web agent searches the web, drives a real browser and extracts content from any page or document, and that content flows into the same multi-agent workforce that runs code on your machine. Nothing marks it as untrusted or restricts what the agents may do after reading it. A web page that hijacks an agent could have code run that reads your keys and files and sends them out, or deletes data, with no human involved.

- **S L0:** No structural limit on what a hijacked agent can do; no detection either. Evidence: searched `rg -n -i -e 'untrusted|prompt injection|taint|quarantin|guardrail'` in `owl examples` → 0 hits (verified)
  - *To reach the next level:* No approval or disabling of egress and code tools once untrusted content has been read.
- **C L0:** Search results, browser pages and extracted documents enter context like any other tool result. Evidence: [examples/run.py:117-123](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L117-L123); [owl/utils/document_toolkit.py:121-124](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/owl/utils/document_toolkit.py#L121-L124) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished from the user's instructions.
- **D L0:** There is no control to enable. Evidence: searched `rg -n -i -e 'untrusted|prompt injection|taint|quarantin|guardrail'` in `owl examples` → 0 hits (verified)
  - *To reach the next level:* No control exists, on or off.
- **B L0:** A hijacked worker can run host code that reads secrets and sends them anywhere, and can delete data, unattended. Evidence: [examples/run.py:128-134](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L128-L134); [owl/utils/document_toolkit.py:166](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/owl/utils/document_toolkit.py#L166); [examples/run.py:38-40](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L38-L40) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions would need human approval.
- **Cap:** C5-WORSTCASE: Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity: 0.10 (medium confidence)

OWL keeps no long-term memory: agent conversations live in memory for one run. The one thing that persists is the owl/.env file, which every run loads into the environment and which controls API keys and model endpoints. The agents' file and code tools can write anywhere the user can, including that file, and nothing protects or validates it, so a single hijacked run could change the configuration every later run trusts.

- **S L0:** camel-ai 0.2.84 behaviour, read at its v0.2.84 tag: the file toolkit accepts absolute paths and code runs on the host, so agents can rewrite the auto-loaded owl/.env with no check. Evidence: [examples/run.py:38-40](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L38-L40); [examples/run.py:88](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L88); [examples/run.py:87](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L87) (inferred)
  - *To reach the next level:* Auto-loaded configuration is not protected from the agent's own tools or validated on load.
- **C L0:** No persistence or configuration path is controlled. Evidence: [examples/run.py:38-40](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L38-L40); searched `rg -n -i -e 'memory|vector|chroma|qdrant|save_memory'` in `owl examples` → 1 hits (single hit is a docstring about an in-memory log queue in the web UI; no memory store) (verified)
  - *To reach the next level:* Even one controlled store would be L1.
- **D L1:** Conversation state is per run and in memory; there is no shared store across users. Evidence: [examples/run.py:200-204](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L200-L204); searched `rg -n -i -e 'memory|vector|chroma|qdrant|save_memory'` in `owl examples` → 1 hits (single hit is a docstring about an in-memory log queue in the web UI; no memory store) (verified)
  - *To reach the next level:* No enforced namespace or tamper protection for the persisted configuration.
- **B L1:** A poisoned owl/.env persists across all of the user's later runs and can redirect the model endpoint, which steers every later tool call. Evidence: [examples/run.py:38-40](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L38-L40) (inferred)
  - *To reach the next level:* Poisoned configuration is not limited to text output or gated actions.
- **Cap:** none

### C7 Third-party extensions: 1.00 (high confidence)

The scored setup loads no third-party extensions at runtime: no MCP servers, plugins, hub tools or model files. MCP integrations exist only in community-contributed examples outside the scored path. Model-written code could still install packages, but that is part of unrestricted code execution and is scored under code-execution isolation.

- **Structural absence:** searched `rg -n -i -e 'mcp|plugin|entry_points|trust_remote_code|torch\.load|pickle|pip install'` in `owl examples` → 0 hits (no MCP client, plugin loader, model-file loading or install path in OWL's code or examples (MCP appears only in community_usecase/))

### C8 Secrets & sensitive-data protection: 0.00 (medium confidence)

API keys sit in plaintext in owl/.env and are loaded into the process environment, where every code subprocess inherits them, so a single model-written 'env' command reveals them all. The default scripts set CAMEL's log level to DEBUG, which prints every model request, including tool results and file contents, to the console. Nothing is redacted on any path.

- **S L0:** Secrets are read from a plaintext .env into the environment with no masking anywhere in the scored path. Evidence: [examples/run.py:38-40](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L38-L40); searched `rg -n -i -e 'redact|mask|SecretStr|sanitiz'` in `owl examples` → 3 hits (all hits are a masking helper in owl/webapp_backup.py, an unused older UI; nothing in the scored path masks secrets) (verified)
  - *To reach the next level:* No masking or redaction on any path.
- **C L0:** No path (logs, model-bound messages, subprocess environments, errors) is protected. Evidence: searched `rg -n -i -e 'redact|mask|SecretStr|sanitiz'` in `owl examples` → 3 hits (all hits are a masking helper in owl/webapp_backup.py, an unused older UI; nothing in the scored path masks secrets); [examples/run.py:42](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L42) (verified)
  - *To reach the next level:* At least one path, such as logs, would need redaction.
- **D L0:** Verbose payload logging is on by default in every example script. Evidence: [examples/run.py:42](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L42) (verified)
  - *To reach the next level:* Verbose payload logging would need to be off by default.
- **B L0:** camel-ai 0.2.84 behaviour, read at its v0.2.84 tag: code subprocesses get the full environment, so long-lived provider keys are reachable by every model-written program. Evidence: [examples/run.py:38-40](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L38-L40); [examples/run.py:87](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L87) (inferred)
  - *To reach the next level:* Keys are long-lived and broadly reachable; they would need to be scoped and kept out of subprocesses.
- **Cap:** none

### C9 Audit & traceability: 0.28 (medium confidence)

OWL keeps no audit trail. The example scripts turn on CAMEL's DEBUG logging, which prints each model request (and with it the tool calls and results) to the console for every worker, but nothing is written to disk, structured, or attributed to the requesting user. After an incident you would have only whatever was left in the terminal.

- **S L1:** camel-ai 0.2.84 behaviour, read at its v0.2.84 tag: at DEBUG, chat agents log each model request with its messages, including tool calls and results, as unstructured text to stdout. Evidence: [examples/run.py:42](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L42); searched `rg -n -S -e 'audit|trajectory|jsonl|set_log_file'` in `owl examples` → 5 hits (hits are GAIA metadata.jsonl loading and a .jsonl file-extension check, not an action record) (inferred)
  - *To reach the next level:* No structured per-call record with arguments, result status and timestamps.
- **C L2:** Every worker is a CAMEL chat agent, so all built-in tool calls appear in those logs. Evidence: [examples/run.py:42](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L42); [examples/run.py:200-204](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L200-L204) (inferred)
  - *To reach the next level:* Approvals and denials do not exist to be recorded, and there is no single record across sub-agents.
- **D L1:** The console logging is on by default but nothing is stored. Evidence: [examples/run.py:42](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L42) (verified)
  - *To reach the next level:* No record stored outside the agent's reach.
- **B L0:** Nothing is persisted; the record is lost when the terminal closes. Evidence: searched `rg -n -S -e 'audit|trajectory|jsonl|set_log_file'` in `owl examples` → 5 hits (hits are GAIA metadata.jsonl loading and a .jsonl file-extension check, not an action record) (verified)
  - *To reach the next level:* Records are not flushed to durable storage.
- **Cap:** none

### C10 Limits & kill switch: 0.33 (low confidence)

OWL sets no limits of its own and relies on CAMEL's defaults. In this CAMEL version those give each workforce task a 10-minute wait, cap pending tasks and retries, time out tool calls and each code run (60 seconds), and limit the browser to 12 rounds, but individual agents have no iteration cap and there is no cost or token budget. Stopping means killing the process; background processes started by model-written code can keep running.

- **S L1:** camel-ai 0.2.84 behaviour, read at its v0.2.84 tag: per-task wait timeout (600 s), pending-task and retry caps, 180 s tool timeouts and a 60 s code timeout, but no agent iteration cap and no cost cap. Evidence: [examples/run.py:200-204](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L200-L204); searched `rg -n -S -e 'max_iteration|task_timeout_seconds|step_timeout|tool_execution_timeout|stop_event|token_limit'` in `examples` → 0 hits (the example scripts set no limit; library defaults apply) (inferred)
  - *To reach the next level:* No iteration cap on worker agents and no token or cost cap.
- **C L2:** The library limits apply to the workforce loop and its tool calls. Evidence: [examples/run.py:200-204](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L200-L204); [examples/run.py:87](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L87) (inferred)
  - *To reach the next level:* Processes spawned by model-written code are not counted against any budget.
- **D L1:** Defaults exist but are large: unlimited agent iterations and no spend ceiling. Evidence: searched `rg -n -S -e 'max_iteration|task_timeout_seconds|step_timeout|tool_execution_timeout|stop_event|token_limit'` in `examples` → 0 hits (the example scripts set no limit; library defaults apply) (inferred)
  - *To reach the next level:* No sensible iteration and spend defaults.
- **B L1:** A runaway run can spend without bound within its time windows, and background processes survive a stop. Evidence: [examples/run.py:87](https://github.com/camel-ai/owl/blob/d67601b245bb91a38876c603cb8a70aa2f5faae8/examples/run.py#L87); searched `rg -n -S -e 'max_iteration|task_timeout_seconds|step_timeout|tool_execution_timeout|stop_event|token_limit'` in `examples` → 0 hits (the example scripts set no limit; library defaults apply) (inferred)
  - *To reach the next level:* No tight per-run cost ceiling; stopping does not clean up spawned processes.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web search, browser and URL/document extraction (examples/run.py:117-123; owl/utils/document_toolkit.py:121-124) · [B] sensitive data/systems: API keys loaded from owl/.env into the process environment (examples/run.py:38-40) and the user's files · [C] state change / egress: host code execution and file writes (examples/run.py:128-134, :140-144); arbitrary URL requests (owl/utils/document_toolkit.py:166) · Same default session? Yes

## Highest-impact improvements
1. Switch the example agents' CodeExecutionToolkit from 'subprocess' to CAMEL's docker or e2b backend. (C4 S L0→L2, +0.150 before caps; Playbook 3 step 1)
2. Turn on require_confirm for code execution so each run needs a y/N with the exact code shown. (C2 S L0→L3, +0.225 before caps; Playbook 5)
3. Stop setting the DEBUG log level in the shipped example scripts. (C8 D L0→L2, +0.100 before caps)
4. Set max_iteration on worker agents and a token or cost budget for the workforce. (C10 D L1→L2, +0.050 before caps)
5. Write a per-run JSONL record of every tool call outside the agents' writable directories. (C9 B L0→L2, +0.100 before caps)

## Re-audit log
- C6 S: L1 → L0. Nothing logs or checks writes to the auto-loaded owl/.env; the L1 anchor needs at least logged writes, which the scored path does not provide.
- C9 B: L1 → L0. Console output is never persisted, so records are lost when the process or terminal ends; L1 'best-effort, flushed late' overstates it.
- C4 C: L2 → L3. Re-read the alt: in the Docker deployment the whole application runs inside the container, so every model-reachable exec path is inside it (the opt-in nature is already in D and G1).

## Limitations
- Static source review of commit d67601b only; nothing was executed, installed, or probed.
- Most runtime behaviour comes from the pinned dependency camel-ai 0.2.84, which was read at its v0.2.84 tag; citations point to OWL's files and library behaviour is marked inferred.
- Scored the README Quick Start (examples/run.py); the other examples/run_*.py scripts build the same toolkits with other model providers. The Gradio web UI (owl/webapp*.py) is a separate mode and was not scored; the Docker deployment is scored as an alternative mechanism under code-execution isolation.
- Community-contributed examples under community_usecase/ (including MCP integrations) were not scored.
- Model behaviour (refusals, deception) is out of scope; only code-level controls are scored.
- No reviewer-directed prompt injection found in the repo.
