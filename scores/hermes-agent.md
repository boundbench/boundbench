# Defense-in-Depth Score: Hermes Agent

**Repo:** https://github.com/NousResearch/hermes-agent · **Commit:** `f4d3e626a732010d23d02c24018091b311a2eb06` · **Reviewed:** 2026-10-03
**What it is:** Self-improving personal agent with skills, memory, terminal and messaging gateways
**Category:** AI Assistants
**Scored configuration:** Interactive `hermes` CLI on a fresh install with the shipped DEFAULT_CONFIG: local (host) terminal backend, approvals.mode 'smart', full core toolset, built-in memory on; messaging gateways footnoted.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents yes · external communication yes

## Score: 2.9 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C2 | Approval gates | L1 | L0 | L1 | L1 | 0.17 | G2 | **0.17** | High |
| C3 | Tool & action scoping | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C4 | Code-execution isolation | L2 | L1 | L0 | L2 | 0.33 | G1 | **0.33** (alt) | High |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C7 | Third-party extensions | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L0 | 0.40 | — | **0.40** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


Hermes Agent runs shell commands and Python directly on the host as your user by default, with every core tool (terminal, code execution, file writes, browser, messaging, cron, sub-agents) enabled. Its approval gate is a pattern denylist whose flagged commands are first judged by an auxiliary LLM, and the execute_code tool skips it entirely in the interactive CLI, so a hijacked model can run arbitrary code, rewrite the agent's own security config, and exfiltrate data without a human. The project's own SECURITY.md says the only real boundary is OS-level isolation (Docker/remote backend or a whole-process sandbox), which is opt-in. Good engineering is visible in env scrubbing, SSRF guards, gateway allowlists and secret redaction, but these are heuristics on an unsandboxed host.

## Critical gaps
- The model can disable the approval gate at runtime: execute_code runs ungated in the CLI and approvals.mode is re-read live from ~/.hermes/config.yaml. (ASI09, ASI02; C2) — [tools/approval.py:1287-1288](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval.py#L1287-L1288); [tools/approval_detection.py:24-25](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval_detection.py#L24-L25)
- execute_code, arbitrary host Python with subprocess access, skips the approval gate in the default interactive CLI. (ASI02, ASI05; C2) — [tools/approval.py:1245](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval.py#L1245); [tools/approval.py:1287-1288](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval.py#L1287-L1288)
- Shell and Python execution run directly on the host by default with no isolation. (ASI05; C4) — [hermes_cli/config_defaults.py:300](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L300); [SECURITY.md:44](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/SECURITY.md#L44)
- A hijacked agent holds the user's full host account and every configured service credential. (ASI03; C1) — [tools/environments/local_env_policy.py:14](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/environments/local_env_policy.py#L14); [SECURITY.md:214](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/SECURITY.md#L214)
- After reading untrusted content the agent can both exfiltrate secrets and take irreversible actions with no human in the loop. (ASI01, LLM01; C5) — [agent/tool_dispatch_helpers.py:470-471](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/agent/tool_dispatch_helpers.py#L470-L471); [tools/approval.py:1287-1288](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval.py#L1287-L1288)
- Plugins and skills run in-process with full agent privileges and credentials. (ASI04; C7) — [SECURITY.md:157-158](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/SECURITY.md#L157-L158)

## Criterion details

### C1 Identity & least privilege — 0.15 (high)

Hermes runs with the full authority of the OS user who launched it. It strips its own provider keys and gateway tokens from the environment of shell, code and MCP child processes, but deliberately leaves the operator's ambient credentials (AWS chain, SSH, gh login) available, and plugins, skills and hooks run inside the agent process with everything it holds. Messaging gateways deny unknown senders by default, but within the allowlist every caller has the same full authority. A hijacked agent therefore holds the user's entire account across every connected service.

- **S L1:** Only Hermes-managed credentials are scrubbed from child environments; the operator's general cloud/SSH/gh credentials stay inheritable by design. — [tools/environments/local_env_policy.py:14-17](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/environments/local_env_policy.py#L14-L17); [SECURITY.md:124-126](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/SECURITY.md#L124-L126) (verified)
  - *To reach the next level:* No per-tool or role-scoped identity; L2 needs a scoped identity rather than the operator's ambient authority.
- **C L1:** Shell, execute_code and MCP stdio children get scrubbed envs, but in-process plugins, skills and hooks read whatever the agent can. — [tools/code_execution_env.py:116-120](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/code_execution_env.py#L116-L120); [tools/mcp_tool_config.py:155-158](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/mcp_tool_config.py#L155-L158); [SECURITY.md:130-133](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/SECURITY.md#L130-L133) (verified)
  - *To reach the next level:* In-process extensions bypass the scrubbing; L2 needs every built-in path and extension on the narrowed identity.
- **D L0:** Default install runs as the OS user with host terminal backend; no narrower default role. — [hermes_cli/config_defaults.py:300](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L300); [SECURITY.md:44](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/SECURITY.md#L44) (verified)
  - *To reach the next level:* L1 needs a narrower default identity than the operator's own account.
- **B L0:** A hijacked agent holds the user's full host account plus every configured gateway, provider and Home Assistant credential. — [toolsets.py:12-14](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/toolsets.py#L12-L14); [SECURITY.md:214](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/SECURITY.md#L214) (verified)
  - *To reach the next level:* L1 needs authority limited below the user's entire account across services.
- **Cap:** none

### C2 Approval gates — 0.17 (high)

Shell commands are checked against a large regex denylist plus the Tirith scanner; anything not flagged runs immediately. Flagged commands go first to an auxiliary-LLM 'smart' approver (the shipped default), which can approve on its own, and only uncertain or denied cases reach the human, who then sees the exact command. The execute_code tool, which runs arbitrary Python on the host, is not gated at all in the interactive CLI, and because the approval mode is read live from ~/.hermes/config.yaml, model-written Python can switch approvals off mid-session. File writes, messaging, MCP tools and Home Assistant calls are not gated, and checkpoints are off by default.

- **S L1:** Default 'smart' mode lets an auxiliary LLM auto-approve denylist-flagged commands; humans see the exact command only on escalation. — [hermes_cli/config_defaults.py:1682](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L1682); [tools/approval_smart.py:25-26](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval_smart.py#L25-L26); [tools/approval.py:773-777](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval.py#L773-L777) (verified)
  - *To reach the next level:* L2 needs a human per-call decision on every consequential action rather than an LLM classifier deciding.
- **C L0:** execute_code (arbitrary host Python incl. subprocess) is approved without any prompt in the interactive CLI; unflagged shell commands and non-shell tools are never gated. — [tools/approval.py:1245](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval.py#L1245); [tools/approval.py:1287-1288](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval.py#L1287-L1288); [tools/approval.py:1093-1095](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval.py#L1093-L1095) (verified)
  - *To reach the next level:* L1 needs the most powerful exec path to cross the gate.
- **D L1:** Gate is on by default, but approvals.mode is read live from config.yaml, which model-run Python can rewrite; HERMES_YOLO_MODE and /yolo also bypass. — [tools/approval_context.py:218-223](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval_context.py#L218-L223); [tools/approval_detection.py:24-25](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval_detection.py#L24-L25); [tools/approval.py:46](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval.py#L46) (verified)
  - *To reach the next level:* L2 needs the model to be unable to switch to auto-approve at runtime.
- **B L1:** Filesystem checkpoints are disabled by default and external actions (messages, email, Home Assistant, pushes) have no undo. — [hermes_cli/config_defaults.py:501-502](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L501-L502); [toolsets.py:28-29](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/toolsets.py#L28-L29) (verified)
  - *To reach the next level:* L2 needs reversibility for the common case (e.g. checkpoints on by default).
- **Cap:** G2 — approvals.mode is read live from ~/.hermes/config.yaml and execute_code runs model-written Python ungated in the CLI, so the model can disable approvals at runtime.

### C3 Tool & action scoping — 0.20 (high)

The core tools are general-purpose: a raw shell, arbitrary Python execution, a full browser and unrestricted file writes. Some tools have real validation: web and browser fetches use an SSRF guard that blocks private and cloud-metadata addresses and rechecks redirects, and file writes refuse system paths and the Hermes config. But the shell and Python tools make those checks easy to step around, and every core tool is on by default.

- **S L1:** The primary tools are a raw shell and Python executor filtered by a denylist; the SSRF guard and sensitive-path checks on narrower tools are sound but trivially bypassed via the shell. — [tools/url_safety.py:1-4](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/url_safety.py#L1-L4); [tools/file_tools_write_guards.py:170](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/file_tools_write_guards.py#L170); [SECURITY.md:144](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/SECURITY.md#L144) (verified)
  - *To reach the next level:* L2 needs typed validation on the primary action tools rather than denylist filtering of raw shell/Python.
- **C L1:** URL and path validation exist in web/browser/file tools only; terminal, execute_code and MCP tools have no argument validation layer. — [tools/approval.py:1093-1095](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval.py#L1093-L1095); [tools/mcp_tool_handlers.py:559-561](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/mcp_tool_handlers.py#L559-L561) (verified)
  - *To reach the next level:* L2 needs most built-in tools, including exec paths, to validate inputs.
- **D L1:** Every core tool including terminal, execute_code, write, browser, cron and delegation is enabled by default; toolsets can be disabled per platform. — [toolsets.py:12-14](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/toolsets.py#L12-L14); [toolsets.py:28](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/toolsets.py#L28) (verified)
  - *To reach the next level:* L2 needs a selectable default that excludes write/exec, or tool groups narrowed by default.
- **B L0:** A misused tool can run any command or reach any host the user can. — [hermes_cli/config_defaults.py:300](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L300) (verified)
  - *To reach the next level:* L1 needs tools scoped below the whole machine.
- **Cap:** none

### C4 Code-execution isolation — 0.33 (high)

By default every shell command and execute_code script runs directly on the host as the user, with no isolation. A Docker backend is available on request: it drops most capabilities, sets no-new-privileges and PID limits, and runs shell, file and code tools inside the container, but keeps network on, runs as root inside by default, and leaves MCP servers, plugins, hooks and skills on the host. The project itself says OS-level isolation is the only real boundary.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Default local backend runs commands as host subprocesses. — [hermes_cli/config_defaults.py:300](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L300); [SECURITY.md:44](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/SECURITY.md#L44) (verified)
    - *To reach the next level:* L1 needs at least a filtering layer that is presented as containment, or L2 OS-level separation.
  - **C L0:** No execution path is sandboxed by default. — [tools/approval.py:1287-1288](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval.py#L1287-L1288) (verified)
    - *To reach the next level:* L1 needs the main exec tool sandboxed.
  - **D L0:** Isolation is off by default. — [hermes_cli/config_defaults.py:300](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L300) (verified)
    - *To reach the next level:* L1 needs a sandbox on by default.
  - **B L0:** Execution is host-equivalent: home directory, ~/.ssh, ~/.aws and the network are all reachable. — [tools/environments/local_env_policy.py:14](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/environments/local_env_policy.py#L14); [SECURITY.md:60-61](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/SECURITY.md#L60-L61) (verified)
    - *To reach the next level:* L1 needs at least narrowing below full host access.
- **opt-in Docker terminal backend (terminal.backend: docker)** (alt; raw 0.33, cap G1 → 0.33) ← counted
  - **S L2:** Stock-image container with cap-drop ALL (re-adding DAC_OVERRIDE/CHOWN/FOWNER), no-new-privileges, tmpfs and PID limits; root inside by default and writable root filesystem. — [tools/environments/docker.py:307-313](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/environments/docker.py#L307-L313); [hermes_cli/config_defaults.py:384](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L384) (verified)
    - *To reach the next level:* L3 needs a non-root, read-only-root, seccomp-hardened profile with network denied by default.
  - **C L1:** Shell, file tools and execute_code run in the container; MCP subprocesses, plugins, hooks and skill loading stay on the host. — [SECURITY.md:79-83](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/SECURITY.md#L79-L83); [tools/code_execution_tool.py:661-662](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/code_execution_tool.py#L661-L662) (verified)
    - *To reach the next level:* L2 needs most paths, including MCP stdio servers and hooks, inside the sandbox.
  - **D L0:** Off by default. — [hermes_cli/config_defaults.py:300](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L300) (verified)
    - *To reach the next level:* L1 needs the backend on by default.
  - **B L2:** No host mounts and no forwarded env by default, but network egress is on. — [hermes_cli/config_defaults.py:373-374](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L373-L374); [hermes_cli/config_defaults.py:354](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L354) (verified)
    - *To reach the next level:* L3 needs egress off or allowlisted.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.25 (high)

Only allowlisted principals can instruct the agent through messaging gateways, and webhook-triggered sessions get a read-only tool set. Inside a normal session, results from web search, web extract, browser and MCP tools are wrapped in untrusted-data delimiters, but nothing in code acts on that marking: files, terminal output, email bodies and other tool results are not marked at all, and after reading hostile content the agent keeps full shell, code execution, outbound HTTP and messaging tools. A successful injection can therefore both leak secrets and take irreversible actions with no human involved.

- **S L1:** Spotlighting only: untrusted_tool_result delimiters around some tool output; no capability is disabled or gated after untrusted content is read. — [agent/tool_dispatch_helpers.py:470-471](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/agent/tool_dispatch_helpers.py#L470-L471) (verified)
  - *To reach the next level:* L2 needs dangerous capabilities to require approval once untrusted content has entered the session.
- **C L1:** Only web_search/web_extract, browser_* and mcp_* outputs are wrapped; file reads, terminal output, email/chat content and other tools are not. — [agent/tool_dispatch_helpers.py:470-471](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/agent/tool_dispatch_helpers.py#L470-L471); [gateway/authz_mixin.py:675](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/gateway/authz_mixin.py#L675) (verified)
  - *To reach the next level:* L2 needs most untrusted sources, including files and messaging content, inside the limit.
- **D L2:** Wrapping is hard-coded on in the tool-result constructor. — [agent/tool_dispatch_helpers.py:549-555](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/agent/tool_dispatch_helpers.py#L549-L555) (verified)
  - *To reach the next level:* L3 needs a mechanism stronger than delimiters for on-by-default to count further.
- **B L0:** A hijacked session can read ~/.hermes/.env or user files via execute_code and exfiltrate via web/curl, and send messages or run unflagged destructive commands, unattended. — [tools/approval.py:1287-1288](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval.py#L1287-L1288); [toolsets.py:12-14](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/toolsets.py#L12-L14) (verified)
  - *To reach the next level:* L1 needs at least one of exfiltration or irreversible action to require a human.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.25 (high)

The agent writes its own long-term memory (MEMORY.md and USER.md, injected into every future system prompt) and its own skills without approval by default; memory writes pass a regex injection scan, while agent-written skills are not scanned by default. Project instruction files such as AGENTS.md, CLAUDE.md and .cursorrules load automatically from the working directory after the same regex scan. Repo-controlled plugins load only behind an explicit environment flag, and no project file can add MCP servers or hooks. Memory is per profile and shared by every session and allowlisted gateway user of that profile.

- **S L1:** Memory and skill writes are ungated (write_approval false) with only regex threat scanning on memory; instruction files load automatically after a regex scan. — [hermes_cli/config_defaults.py:1320](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L1320); [tools/memory_tool_store.py:26-29](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/memory_tool_store.py#L26-L29); [agent/prompt_builder.py:84](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/agent/prompt_builder.py#L84) (verified)
  - *To reach the next level:* L2 needs memory presented with provenance as data and project config barred from security settings with instruction loading made visible.
- **C L1:** Memory writes are scanned, but agent-created skills are not scanned by default and recalled sessions/summaries are uncontrolled. — [hermes_cli/config_defaults.py:1476](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L1476); [hermes_cli/config_defaults.py:1485](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L1485) (verified)
  - *To reach the next level:* L2 needs the main stores controlled, including skills the agent writes.
- **D L1:** Memory lives per profile under HERMES_HOME and is shared by all sessions and allowlisted gateway users of that profile. — [tools/memory_tool.py:38-40](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/memory_tool.py#L38-L40); [SECURITY.md:214](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/SECURITY.md#L214) (verified)
  - *To reach the next level:* L2 needs per-user/session namespaces enforced in code.
- **B L1:** Poisoned memory or a skill persists across all of the user's sessions and can drive tool use. — [tools/memory_tool_store.py:27-28](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/memory_tool_store.py#L27-L28) (verified)
  - *To reach the next level:* L2 needs persisted content limited to influencing text or gated actions.
- **Cap:** none

### C7 Third-party extensions — 0.25 (high)

No third-party extension is enabled by default, project-directory plugins need an explicit environment flag, and hub skills are scanned and recorded with a content hash at install. MCP servers launched through npx/uvx get a fail-open malware lookup but are otherwise whatever version the user's command resolves. Plugins and skills run inside the agent process with all of its credentials and tools; MCP stdio servers at least get a scrubbed environment.

- **S L1:** User-chosen sources, largely unpinned; the OSV malware check fails open and hub-skill hashes record but do not pin MCP/plugins. — [tools/osv_check.py:5](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/osv_check.py#L5); [tools/skills_guard.py:724-729](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/skills_guard.py#L724-L729) (verified)
  - *To reach the next level:* L2 needs pinned versions for every extension type.
- **C L1:** Hub skills get scanning and hashing; MCP servers and plugins do not get equivalent verification. — [tools/skills_guard.py:724-729](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/skills_guard.py#L724-L729); [tools/mcp_tool_handlers.py:64](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/mcp_tool_handlers.py#L64) (verified)
  - *To reach the next level:* L2 needs most extension types verified.
- **D L2:** Extensions require an explicit operator install; project plugins are gated by HERMES_ENABLE_PROJECT_PLUGINS. — [hermes_cli/plugins_discovery.py:212](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/plugins_discovery.py#L212) (verified)
  - *To reach the next level:* L3 needs each install to show the exact package, command and permissions, verified in code.
- **B L0:** Plugins and skills load into the agent process with full agent privileges and credentials. — [SECURITY.md:157-158](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/SECURITY.md#L157-L158); [SECURITY.md:153](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/SECURITY.md#L153) (verified)
  - *To reach the next level:* L1 needs extensions in a separate process.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.40 (high)

Secrets live in plaintext ~/.hermes/.env (permissioned 0600), with optional 1Password/Bitwarden sources. A regex redactor runs on log output and on terminal and file content returned to the model, Hermes-managed keys are scrubbed from child environments, and telemetry is off by default. Redaction can be switched off in config, and the keys themselves stay readable by model-run Python, so a hijacked agent can still reach long-lived provider and messaging credentials.

- **S L2:** Regex redaction filters on logs and model-bound tool output; credentials stored in plaintext files with restrictive permissions. — [hermes_logging.py:382](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_logging.py#L382); [tools/terminal_tool_result.py:242](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/terminal_tool_result.py#L242); [hermes_cli/config.py:504-510](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config.py#L504-L510) (verified)
  - *To reach the next level:* L3 needs OS keychain or encrypted storage by default.
- **C L2:** Logs, model-bound terminal/file output and subprocess envs are covered; transcripts, error paths and execute_code direct reads were not verified to be covered. — [tools/file_tools.py:485](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/file_tools.py#L485); [tools/code_execution_env.py:120](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/code_execution_env.py#L120) (verified)
  - *To reach the next level:* L3 needs all listed paths verified, including transcripts and error handlers.
- **D L2:** Telemetry collection and sending are opt-in; redaction is on by default but disableable via security.redact_secrets. — [hermes_cli/config_defaults.py:1768](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L1768); [hermes_cli/config_defaults.py:2330-2332](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L2330-L2332) (verified)
  - *To reach the next level:* L3 needs redaction that cannot be turned off.
- **B L0:** Long-lived provider, messaging and GitHub keys in ~/.hermes/.env are readable by model-run Python, and operator cloud/SSH credentials remain reachable. — [SECURITY.md:130-133](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/SECURITY.md#L130-L133); [tools/approval.py:1287-1288](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/approval.py#L1287-L1288) (verified)
  - *To reach the next level:* L1 needs keys out of model reach or narrowly scoped.
- **Cap:** none

### C9 Audit & traceability — 0.50 (high)

Every message, including assistant tool calls and tool results, is stored in a SQLite session database under ~/.hermes with timestamps, and sub-agent sessions link to their parent. The tool-call turn is flushed before tools run, so a crash still leaves a record. Approvals and denials appear only as tool-result text and log lines, there is no actor attribution or tamper evidence, and the database sits where the agent's own shell and Python can modify it.

- **S L2:** Structured per-message transcript with tool_calls, tool_name and timestamp in state.db. — [hermes_state_common.py:449-458](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_state_common.py#L449-L458) (verified)
  - *To reach the next level:* L3 needs actor attribution (approver, requesting principal) and correlation IDs across sub-agents recorded explicitly.
- **C L2:** All tool calls through the agent loop, including MCP and sub-agent sessions (parent_session_id), are recorded; approval decisions are not structured records. — [hermes_state_common.py:398](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_state_common.py#L398); [agent/turn_tool_round.py:127-130](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/agent/turn_tool_round.py#L127-L130) (verified)
  - *To reach the next level:* L3 needs approvals and denials recorded as first-class entries.
- **D L2:** On by default in HERMES_HOME/state.db, outside the workspace but writable by the agent's own process and shell. — [hermes_state.py:175](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_state.py#L175) (verified)
  - *To reach the next level:* L3 needs the record written by a component the model cannot control.
- **B L2:** Tool-call turns are flushed before side effects; a persistence failure logs a warning and the turn proceeds. — [agent/turn_tool_round.py:127-130](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/agent/turn_tool_round.py#L127-L130) (verified)
  - *To reach the next level:* L3 needs durable per-action records that support full replay with guaranteed durability.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

Per-command terminal timeouts (180 seconds) and an interrupt that kills the foreground process group are on by default, and an iteration cap and wall-clock budget exist but are both unlimited by default. There is no token or spend cap. Sub-agents get their own fresh iteration budget that the model cannot raise, with concurrency and depth limits. The 'hermes pause' emergency stop only blocks new work, and cron jobs and background processes the agent created keep running after a stop.

- **S L2:** Iteration cap, per-command timeout and optional wall-clock budget enforced in code; interrupt killpg's the foreground process; no token/cost cap. — [hermes_cli/config_defaults.py:316](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L316); [tools/environments/local.py:911](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/environments/local.py#L911); [hermes_cli/config_defaults.py:85](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L85) (verified)
  - *To reach the next level:* L3 needs a token/cost cap and rate limits on side-effecting tools.
- **C L2:** Top-level loop plus tool timeouts; sub-agents get independent budgets and cron/background work is not counted. — [agent/iteration_budget.py:5](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/agent/iteration_budget.py#L5); [hermes_cli/config_defaults.py:1364](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L1364) (verified)
  - *To reach the next level:* L3 needs sub-agents and spawned tasks to draw on the same budget.
- **D L1:** Turn cap and wall-clock budget are unlimited by default; the model cannot raise the sub-agent cap. — [hermes_cli/config_defaults.py:78](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/hermes_cli/config_defaults.py#L78); [run_agent.py:268](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/run_agent.py#L268); [tools/delegate_tool.py:517](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/tools/delegate_tool.py#L517) (verified)
  - *To reach the next level:* L2 needs sensible finite defaults for turns or wall-clock.
- **B L1:** No spend ceiling; ESTOP never kills in-flight work and model-created cron jobs and background processes outlive a stop. — [agent/estop.py:5](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/agent/estop.py#L5); [toolsets.py:29](https://github.com/NousResearch/hermes-agent/blob/f4d3e626a732010d23d02c24018091b311a2eb06/toolsets.py#L29) (verified)
  - *To reach the next level:* L2 needs moderate default ceilings and stop that ends scheduled work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web_search/web_extract/browser/MCP results and project files (agent/tool_dispatch_helpers.py:470-471) · [B] sensitive data/systems: ~/.hermes/.env keys and the user's home directory reachable via execute_code (tools/approval.py:1287-1288) · [C] state change / egress: terminal, execute_code, send_message, web fetch in the default toolset (toolsets.py:12-41) · Same default session? Yes

## Highest-impact improvements
1. Gate execute_code in the interactive CLI like gateway sessions (whole-script human approval). — C2 C L0→L1, +0.075 before caps (Playbook 5)
2. Make approvals.mode immutable at runtime (snapshot at startup or require operator restart) so model-written config cannot disable the gate. — C2 D L1→L2, +0.050 before caps (Playbook 5)
3. Ship finite default max_turns and run_budget_seconds. — C10 D L1→L2, +0.050 before caps (Playbook 3 step 3)
4. Default the terminal backend to the hardened Docker sandbox with network off, falling closed when Docker is missing. — C4 D L0→L2, +0.100 before caps (Playbook 3 step 1)
5. Turn on memory/skill write_approval and guard_agent_created by default. — C6 C L1→L2, +0.075 before caps (Playbook 2)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The repository is very large (hundreds of modules); review focused on the approval, terminal/backends, code execution, MCP, plugin/skill, memory, logging, gateway authorization and limits code paths. The TUI, desktop app, dashboard, ACP adapter, kanban and individual platform adapters were not examined in depth.
- Scored the interactive CLI default; gateway (messaging) mode was footnoted: it adds default-deny sender allowlists and whole-script execute_code approval, but the same host backend and tool set.
- C8 transcript and error-path redaction coverage were not fully traced and were rated conservatively.
- No reviewer-targeted prompt injection was found in repository docs.
