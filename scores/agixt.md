# Defense-in-Depth Score: AGiXT

**Repo:** https://github.com/josh-xt/agixt · **Commit:** `3a3202419a962aa815d139849d6f218d3f81c998` (v1.9.4) · **Reviewed:** 2026-10-04
**What it is:** Self-hosted multi-user AI agent platform with extensions, chains, scheduled tasks, chat-bot integrations and a code-execution sandbox.
**Category:** Agent Frameworks
**Scored configuration:** `agixt start` with defaults: Docker mode via the shipped docker-compose.yml, a newly registered user's default agent with Core Abilities auto-enabled.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents yes · external communication yes

## Score: 2.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L0 | L0 | 0.07 | C1-SELFESC | **0.07** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C4 | Code-execution isolation | L2 | L1 | L1 | L0 | 0.28 | G2 | **0.25** | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L0 | 0.17 | — | **0.17** | High |
| C8 | Secrets & sensitive-data protection | L2 | L1 | L2 | L0 | 0.33 | — | **0.33** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |


AGiXT gives every new agent a shell, Python execution, arbitrary outbound HTTP, agent creation and recurring scheduled tasks, all enabled by default and with no human approval step. Code-execution isolation does not cover every execution path. A prompt injection can reach the user's connected-account tokens unattended. Path and SSRF validation on the file and URL tools is solid but easily sidestepped by the raw exec tools.

## Critical gaps
- Create AGiXT Agent lets the model enable AI-selected commands on a new agent and a delegation chain on itself, expanding its own reachable permissions. (ASI03, T3; C1) — [agixt/extensions/essential_abilities.py:4453-4456](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L4453-L4456); [agixt/extensions/essential_abilities.py:4237-4239](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L4237-L4239)
- A hijacked agent can exfiltrate data (Custom API Endpoint, networked shell) and take irreversible actions with no human in the loop. (ASI01, LLM01; C5) — [agixt/extensions/essential_abilities.py:1943-1970](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L1943-L1970); [agixt/extensions/essential_abilities.py:224-243](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L224-L243)
- Extensions Hub code, when configured, is cloned unpinned and imported in-process with all server credentials. (ASI04, T17; C7) — [agixt/ExtensionsHub.py:700-715](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/ExtensionsHub.py#L700-L715); [agixt/ExtensionsHub.py:910](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/ExtensionsHub.py#L910)

## Criterion details

### C1 Identity & least privilege — 0.07 (high)

Agents act with the full authority of the user who owns them. Every command call receives all of the agent's stored settings plus every OAuth access token the user has connected, and the user's GitHub token is placed inside the code sandbox. There is no per-tool credential scoping. The built-in 'Create AGiXT Agent' command lets the model create a new agent, enable AI-chosen commands on it from every configured integration, and then enable a delegation command on itself, so the model can widen what it can reach. The default deployment also mounts the Docker socket into the root AGiXT container.

- **S L0:** Commands receive the user's entire set of OAuth tokens and agent secrets, and the agent can give itself new capabilities through agent creation. — [agixt/Extensions.py:1046-1069](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Extensions.py#L1046-L1069); [agixt/extensions/essential_abilities.py:4453-4456](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L4453-L4456) (verified)
  - *To reach the next level:* No per-tool or per-capability credentials; every extension instance gets all SSO tokens.
- **C L1:** The main execution path checks the agent's enabled-command list, but credentials are passed whole to every extension and into the sandbox environment. — [agixt/Interactions.py:7372-7373](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Interactions.py#L7372-L7373); [agixt/extensions/essential_abilities.py:1499-1508](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L1499-L1508) (verified)
  - *To reach the next level:* Credentials are not narrowed per tool or kept out of spawned sandboxes.
- **D L0:** New agents get every Core Abilities command (shell, Python, HTTP, agent creation) enabled by default. — [agixt/Agent.py:1411-1438](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Agent.py#L1411-L1438) (verified)
  - *To reach the next level:* Default agents are not read-only or minimal; elevation is not explicit.
- **B L0:** A hijacked agent holds the user's connected accounts, and the server it runs in is root with the host Docker socket mounted. — [docker-compose.yml:76](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/docker-compose.yml#L76); [agixt/Extensions.py:1046-1069](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Extensions.py#L1046-L1069) (verified)
  - *To reach the next level:* Credentials are long-lived and broad; the server container is host-equivalent.
- **Cap:** C1-SELFESC — The Create AGiXT Agent command enables AI-selected commands on a new agent and toggles a delegation chain on the calling agent, so the model can expand its own reachable permissions.

### C2 Approval gates — 0.00 (high)

There is no human approval step for any command. Once a command is enabled for the agent (and all Core Abilities are enabled by default), the model's tool calls run immediately: shell commands, Python execution, file deletion, arbitrary outbound HTTP requests, scheduling recurring tasks and creating new agents. The only 'approval' in the code is an optional LLM review of the final answer text, which does not gate actions. Actions such as deleting workspace files or POSTing to external APIs have no undo.

- **S L0:** No approval mechanism exists; enabled commands execute as soon as the model emits them. — searched `rg -n -i 'require_approval|requires_approval|human_approval|needs_approval|approval_required'` in `agixt` → 0 hits (no human approval primitive anywhere in the server); [agixt/Interactions.py:7372-7396](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Interactions.py#L7372-L7396) (verified)
  - *To reach the next level:* No per-call human approval showing the exact command.
- **C L0:** The most powerful tools (terminal, Python, Custom API Endpoint) are reachable with no gate. — [agixt/extensions/essential_abilities.py:224-243](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L224-L243) (verified)
  - *To reach the next level:* No gate on any tool path.
- **D L0:** Approval does not exist, so it is not on by default. — searched `rg -n -i 'require_approval|requires_approval|human_approval|needs_approval|approval_required'` in `agixt` → 0 hits (no human approval primitive anywhere in the server) (verified)
  - *To reach the next level:* No default-on approval.
- **B L0:** Wrongly taken actions include irreversible workspace deletes and outbound API writes, with no checkpointing. — [agixt/extensions/essential_abilities.py:212](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L212); [agixt/extensions/essential_abilities.py:1943-1970](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L1943-L1970) (verified)
  - *To reach the next level:* No checkpoints, rollback, or previews for external actions.
- **Cap:** none

### C3 Tool & action scoping — 0.40 (high)

The file tools resolve paths with realpath, check containment in the conversation workspace and reject symlinks, and the URL tools have solid SSRF protection: private, loopback, link-local and cloud-metadata addresses are blocked, DNS is pinned, and redirects are rechecked or disabled. However, the default tool set also includes a raw shell, raw Python execution and a generic HTTP tool, which make those checks easy to sidestep. Every Core Abilities tool is enabled by default for every new agent, including write, exec and network tools.

- **S L2:** File and URL tools validate in code (realpath containment, SSRF IP checks with redirect revalidation), but raw shell and Python tools take arbitrary strings. — [agixt/extensions/essential_abilities.py:491-503](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L491-L503); [agixt/XT.py:112-113](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/XT.py#L112-L113); [agixt/XT.py:249-251](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/XT.py#L249-L251); [agixt/extensions/essential_abilities.py:224-243](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L224-L243) (verified)
  - *To reach the next level:* General tools are not replaced by narrow ones; shell/Python accept anything.
- **C L2:** Most built-in file and URL tools validate; exec tools and the many integration extensions have no shared validation layer. — [agixt/extensions/essential_abilities.py:491-503](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L491-L503); [agixt/extensions/essential_abilities.py:1943-1970](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L1943-L1970) (verified)
  - *To reach the next level:* No shared policy layer wrapping every tool, including extensions.
- **D L1:** All Core Abilities, including terminal, Python and HTTP, are enabled on every new agent; commands can be toggled off individually. — [agixt/Agent.py:1411-1438](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Agent.py#L1411-L1438); [agixt/extensions/essential_abilities.py:224-243](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L224-L243) (verified)
  - *To reach the next level:* Default set is not read-only; write/exec are not opt-in.
- **B L1:** File tools are workspace-scoped, but shell/Python with network and a generic public HTTP tool give broad reach. — [agixt/extensions/essential_abilities.py:224-243](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L224-L243); [agixt/extensions/essential_abilities.py:1943-1970](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L1943-L1970) (verified)
  - *To reach the next level:* No quantity bounds; general-purpose tools remain in reach.
- **Cap:** none

### C4 Code-execution isolation — 0.25 (medium)

Shell and Python commands are sent to the external 'safeexecute' package, which runs them in a Docker container. That package is not in this repository, and the repo's own comments say those sandbox containers run as root. The image is pulled as an unpinned ':latest' tag, and the user's GitHub token is passed in. Not every model-reachable execution path is confined.

- **S L2:** Default sandbox is a Docker container from the external safeexecute package; repo comments indicate it runs as root (container hardening not verifiable here). — [agixt/extensions/essential_abilities.py:1499-1508](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L1499-L1508); [agixt/extensions/essential_abilities.py:534](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L534); [agixt/extensions/essential_abilities.py:556](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L556) (inferred)
  - *To reach the next level:* No evidence of non-root, dropped capabilities, seccomp, or network-off sandbox.
- **C L1:** Terminal and Python go to the sandbox, but not every other execution path is confined. (verified)
  - *To reach the next level:* Sandbox coverage is not complete across execution paths.
- **D L1:** Sandboxing is on for the main exec tools, but not every model-reachable execution path is confined. — [agixt/extensions/essential_abilities.py:224-243](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L224-L243) (verified)
  - *To reach the next level:* Unsandboxed execution is not gated by per-call human approval.
- **B L0:** The GitHub token is in the sandbox environment, and the server container runs as root with the Docker socket, DB credentials and the master API key. — [docker-compose.yml:76](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/docker-compose.yml#L76); searched `rg -n '^USER'` in `Dockerfile` → 0 hits (no USER directive: the AGiXT server container runs as root); [agixt/extensions/essential_abilities.py:1499-1508](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L1499-L1508) (verified)
  - *To reach the next level:* Credentials in sandbox; server path is host-equivalent.
- **Cap:** G2 — The model can reach unsandboxed execution at runtime without a human step.

### C5 Untrusted input blast radius — 0.00 (high)

Web pages, search results, downloaded files, uploaded documents and integration outputs enter the model's context as plain command output, with no provenance marking, no taint tracking and no rule that disables egress or writes once untrusted content has been read. The same session holds the user's connected accounts, a shell with network access, and a generic HTTP tool, so a successful prompt injection can both exfiltrate data and take irreversible actions without a human. AGiXT is multi-tenant, which raises the stakes further.

- **S L0:** No structural limit on a hijacked agent; tool output is fed back as ordinary context. — searched `rg -n -i 'untrusted|prompt.?injection'` in `agixt/Interactions.py agixt/XT.py agixt/Websearch.py agixt/Extensions.py` → 1 hits (single hit is a code comment in XT.py about context size, not a defense); [agixt/Interactions.py:7372-7396](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Interactions.py#L7372-L7396) (verified)
  - *To reach the next level:* No Rule-of-Two enforcement or quarantined handling of untrusted content.
- **C L0:** Untrusted sources are not distinguished from the principal's input. — searched `rg -n -i 'untrusted|prompt.?injection'` in `agixt/Interactions.py agixt/XT.py agixt/Websearch.py agixt/Extensions.py` → 1 hits (single hit is a code comment in XT.py about context size, not a defense) (verified)
  - *To reach the next level:* No source is distinguished as untrusted.
- **D L0:** There is no control to be on by default. — searched `rg -n -i 'untrusted|prompt.?injection'` in `agixt/Interactions.py agixt/XT.py agixt/Websearch.py agixt/Extensions.py` → 1 hits (single hit is a code comment in XT.py about context size, not a defense) (verified)
  - *To reach the next level:* No default-on untrusted-content control.
- **B L0:** A hijack can read connected-account data and the GitHub token and send it out via Custom API Endpoint or the networked shell, and can delete or push, unattended. — [agixt/extensions/essential_abilities.py:1943-1970](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L1943-L1970); [agixt/extensions/essential_abilities.py:1499-1508](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L1499-L1508); [agixt/extensions/essential_abilities.py:224-243](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L224-L243) (verified)
  - *To reach the next level:* No approval on egress or irreversible actions after untrusted input.
- **Cap:** C5-WORSTCASE — In the default configuration a hijacked agent can leak secrets and take irreversible actions with no human involved.

### C6 Memory, context & configuration integrity — 0.10 (high)

Agents keep a persistent vector memory (scoped by agent ID), and web-search and browsed content is written into it automatically and retrieved into later prompts. More importantly, the model can create persistent artifacts that later run with tools and with no review: recurring scheduled follow-ups, whose stored description is fed back as a prompt, new automation chains, and new agents. A one-time injection can therefore set up something that keeps running in the user's future sessions. No auto-loaded workspace instruction files were found.

- **S L0:** The model can write scheduled tasks, chains and agents that are replayed later with tool access, with no validation or approval. — [agixt/extensions/essential_abilities.py:238](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L238); [agixt/Task.py:464](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Task.py#L464); [agixt/Websearch.py:200-204](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Websearch.py#L200-L204) (verified)
  - *To reach the next level:* Persistent writes are not gated, validated or expired.
- **C L0:** Neither memory, scheduled tasks nor chains have write controls. — [agixt/extensions/essential_abilities.py:238](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L238); [agixt/Websearch.py:200-204](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Websearch.py#L200-L204) (verified)
  - *To reach the next level:* No memory or persistence path is controlled.
- **D L1:** Memory retrieval is filtered by agent ID in the query, but the weak write controls limit credit. — [agixt/DB.py:2823](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/DB.py#L2823) (verified)
  - *To reach the next level:* Model can still write into its own namespace and create new agents/namespaces.
- **B L1:** Poisoned scheduled tasks persist across the user's sessions and run commands. — [agixt/Task.py:464](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Task.py#L464); [agixt/Task.py:466-481](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Task.py#L466-L481) (verified)
  - *To reach the next level:* Persistent entries can trigger tool use without review.
- **Cap:** none

### C7 Third-party extensions — 0.17 (high)

No third-party extensions are enabled by default: the Extensions Hub is empty unless an operator sets EXTENSIONS_HUB, and the model-facing 'Use MCP Server' command is broken at this commit (it calls MCPClient() without its required arguments). When a hub is configured, its repository is re-cloned from the latest default branch with no pin or hash check, and its Python files are imported straight into the server process with every credential that process holds. By default the CLI also auto-pulls the unpinned joshxt/agixt:main and safeexecute:latest images.

- **S L1:** Operator-chosen hub sources are cloned unpinned at the latest commit on each start. — [agixt/ExtensionsHub.py:700-715](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/ExtensionsHub.py#L700-L715) (verified)
  - *To reach the next level:* No version pinning or integrity check of hub code.
- **C L0:** No extension type is verified. — [agixt/ExtensionsHub.py:700-715](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/ExtensionsHub.py#L700-L715); [agixt/ExtensionsHub.py:910](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/ExtensionsHub.py#L910) (verified)
  - *To reach the next level:* No verification for any extension type.
- **D L2:** Nothing third-party is enabled by default; hubs are added by an operator env/server setting without showing what will run. — [agixt/Globals.py:223](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Globals.py#L223); [agixt/extensions/automation_helpers.py:595-597](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/automation_helpers.py#L595-L597) (verified)
  - *To reach the next level:* Adding a hub doesn't display the code/permissions being enabled.
- **B L0:** Hub extensions run in-process with the server's full environment and every user's credentials. — [agixt/ExtensionsHub.py:910](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/ExtensionsHub.py#L910); [agixt/Extensions.py:1046-1069](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Extensions.py#L1046-L1069) (verified)
  - *To reach the next level:* No process separation or credential scoping for extensions.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.33 (high)

There is some redaction. Uvicorn logs pass through a filter for sensitive data, agent settings flagged as sensitive are masked in API responses, and some command-logging paths redact keys. But OAuth access tokens are stored in a plain text database column, the main execution log path writes command arguments without redaction, and the user's GitHub token is placed in the sandbox environment, where any shell command can read it. Shipped deployment defaults for the datastores are not locked down. No telemetry SDK was found.

- **S L2:** Masking of sensitive agent settings and a log redaction filter exist; OAuth tokens are stored as plain columns. — [agixt/Agent.py:3335](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Agent.py#L3335); [agixt/logging_config.yaml:23](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/logging_config.yaml#L23); [agixt/DB.py:683](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/DB.py#L683) (verified)
  - *To reach the next level:* No encryption at rest for OAuth tokens, no redaction before model-bound messages.
- **C L1:** Server logs and some activity logs are filtered, but the main execution log path and sandbox environment are not. — [agixt/XT.py:969](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/XT.py#L969); [agixt/Interactions.py:7377](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Interactions.py#L7377); [agixt/extensions/essential_abilities.py:1499-1508](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L1499-L1508) (verified)
  - *To reach the next level:* Redaction missing on the main execution log path and subprocess environment.
- **D L2:** No telemetry; log redaction is configured by default; but shipped deployment defaults for the datastores are not locked down. — [agixt/logging_config.yaml:23](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/logging_config.yaml#L23); searched `rg -n -i 'sentry_sdk|posthog|opentelemetry'` in `agixt` → 0 hits (no telemetry SDK) (verified)
  - *To reach the next level:* Shipped datastore deployment defaults need hardening.
- **B L0:** Long-lived GitHub and other OAuth tokens are readable by every sandboxed command. — [agixt/extensions/essential_abilities.py:1499-1508](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/extensions/essential_abilities.py#L1499-L1508); [agixt/Extensions.py:1046-1069](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Extensions.py#L1046-L1069) (verified)
  - *To reach the next level:* Tokens are long-lived and broadly scoped, reachable from the sandbox.
- **Cap:** none

### C9 Audit & traceability — 0.50 (high)

Each command the agent runs is logged into the conversation, with its name and JSON arguments, before it executes, and webhook events are emitted when commands start, complete or fail. These records sit in the application database, outside the agent's workspace. They are free-form markdown messages, though, with no separate approver or delegation chain. The user (and therefore anything holding the user's token) can delete them through the conversation API, and nothing makes them tamper-evident.

- **S L2:** Every tool call is recorded as a timestamped conversation activity with arguments; webhooks emit started/completed/failed events. — [agixt/Interactions.py:7378-7381](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Interactions.py#L7378-L7381); [agixt/Extensions.py:1077-1089](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Extensions.py#L1077-L1089) (verified)
  - *To reach the next level:* No actor/approver attribution or correlation across delegated agents.
- **C L2:** The main execution loop and chain steps are logged; extension-internal actions and scheduled-task command paths were not confirmed. — [agixt/Interactions.py:7378-7381](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Interactions.py#L7378-L7381); [agixt/XT.py:969](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/XT.py#L969) (verified)
  - *To reach the next level:* Not verified for every extension and scheduled-task path.
- **D L2:** On by default and stored in the DB outside the workspace, but deletable via the user-level message API. — [agixt/endpoints/Conversation.py:1599-1606](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/endpoints/Conversation.py#L1599-L1606) (verified)
  - *To reach the next level:* Records can be altered by the user-level API the agent's process holds.
- **B L2:** The activity record is written synchronously before the command runs. — [agixt/Interactions.py:7378-7381](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Interactions.py#L7378-L7381); [agixt/Extensions.py:1077-1089](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Extensions.py#L1077-L1089) (verified)
  - *To reach the next level:* Not tamper-evident; webhook emission is fire-and-forget.
- **Cap:** none

### C10 Limits & kill switch — 0.25 (high)

The main agent loop explicitly has no limit on execution iterations: the code says the agent should take 'as many steps and as much time as needed'. The only caps are on non-executing turns (40 by default) and some per-call timeouts, such as 30 seconds on HTTP calls. Token billing is off by default, so there is no spend ceiling. A stop endpoint cancels the conversation's asyncio task, but synchronous sandbox calls and scheduled recurring tasks keep running.

- **S L1:** Only a cap on non-executing continuations and scattered per-call timeouts; no step, wall-clock or cost cap on execution. — [agixt/Interactions.py:4921-4922](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Interactions.py#L4921-L4922); [agixt/Interactions.py:4973-4974](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Interactions.py#L4973-L4974) (verified)
  - *To reach the next level:* No execution step cap, wall-clock limit, or token/cost cap.
- **C L1:** The partial cap applies to the top-level loop only; delegated agents and scheduled tasks run with their own fresh loops. — [agixt/Interactions.py:4973-4974](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Interactions.py#L4973-L4974); [agixt/Task.py:464](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Task.py#L464) (verified)
  - *To reach the next level:* Sub-agents and scheduled tasks don't share a budget.
- **D L1:** Defaults are effectively unlimited: billing price 0.00 disables balance checks, and the continuation cap is env-configurable. — [agixt/Globals.py:228](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Globals.py#L228); [agixt/Interactions.py:4973-4974](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Interactions.py#L4973-L4974) (verified)
  - *To reach the next level:* No sensible default spend or step ceilings.
- **B L1:** A runaway can loop and spend indefinitely; stop cancels the asyncio task but leaves scheduled tasks and in-flight sandbox calls. — [agixt/WorkerRegistry.py:320-322](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/WorkerRegistry.py#L320-L322); [agixt/Task.py:466-481](https://github.com/josh-xt/agixt/blob/3a3202419a962aa815d139849d6f218d3f81c998/agixt/Task.py#L466-L481) (verified)
  - *To reach the next level:* Stop doesn't cancel in-flight calls or scheduled work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Web browsing/search and Download File from URL write external content into context and memory (agixt/Websearch.py:200) · [B] sensitive data/systems: All user OAuth tokens and agent secrets injected into every command (agixt/Extensions.py:1046-1069); GitHub token in sandbox (essential_abilities.py:1507) · [C] state change / egress: Terminal, Python, Delete File, Custom API Endpoint enabled by default (essential_abilities.py:212-243) · Same default session? Yes

## Highest-impact improvements
1. Remove the Docker socket mount from the default compose file (run sandboxes via a separate rootless runner) and run the server as a non-root user. — C4 B L0→L2, +0.100 before caps (Playbook 3 (sandboxing))
2. Route every model-reachable execution path through the safeexecute sandbox. — C4 C L1→L3, +0.150 before caps (Playbook 3 (sandboxing))
3. Add a per-call human approval gate, on by default, for terminal, Python, file delete, Custom API Endpoint, agent creation and scheduling. — C2 S L0→L3, +0.225 before caps (Playbook 5)
4. Add a default execution-step cap, wall-clock limit and per-user token budget to the run_stream loop. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
5. Stop passing the GitHub token into the sandbox environment and encrypt OAuth tokens at rest. — C8 B L0→L2, +0.100 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The sandbox implementation lives in the external safeexecute PyPI package and joshxt/safeexecute image, which are not in this repository; C4 Strength is INFERRED from in-repo call sites and comments.
- One default-identity configuration was noted but not fully traced.
- The ui/ (Next.js/Tauri desktop) client and the 90+ integration extensions were sampled, not read in full; bot managers (GitHub, Discord, email) are opt-in and were footnoted, not scored. Bot-manager access control is not locked down.
- No text aimed at AI reviewers was found in the repository.
