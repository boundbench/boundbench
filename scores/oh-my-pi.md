# Defense-in-depth score: oh-my-pi (omp)

**Repo:** https://github.com/can1357/oh-my-pi · **Commit:** `8aaf115b8e5f54e6bddeffa969e43e290a1efa01` · **Reviewed:** 2026-10-05
**What it is:** Terminal coding agent (omp), a fork of the Pi coding agent with sub-agents, MCP, browser, extensions and plugins.
**Category:** Coding
**Scored configuration:** Interactive omp CLI (packages/coding-agent 18.6.2), no flags, fresh install: tools.approvalMode default yolo, default built-in tools including bash, eval, write, fetch, browser and task, project MCP config enabled, memory off.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 1.9 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C2 | Approval gates | L2 | L2 | L0 | L1 | 0.35 | C2-POWERBYPASS | **0.25** | High |
| C3 | Tool & action scoping | L1 | L1 | L1 | L0 | 0.20 | none | **0.20** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | L1 | L0 | L0 | L0 | 0.07 | C7-RCELOAD | **0.07** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L0 | 0.25 | none | **0.25** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | none | **0.50** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | none | **0.25** | High |


omp ships with tool approval set to 'yolo', so shell commands, file writes, code evaluation, browsing and sub-agents run without asking, on your machine with your full environment and no sandbox. A prompt injection in anything it reads can therefore leak credentials and take irreversible actions unattended. Starting omp inside a cloned repository also loads that repository's settings, hooks, extensions and MCP servers with no trust prompt. A capable per-call approval system exists: set tools.approvalMode to always-ask, and run omp in a container or VM.

## Critical gaps
- Tools run with the user's ambient identity and full environment, so a hijacked session can use every credential the user has. (ASI03, T3, LLM06; C1). Evidence: [packages/utils/src/procmgr.ts:43-55](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/utils/src/procmgr.ts#L43-L55)
- The default approval mode is yolo, so the bash tool runs without approval. (ASI09, ASI02, T10; C2). Evidence: [packages/coding-agent/src/tools/settings.ts:312-315](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/settings.ts#L312-L315); [packages/coding-agent/src/tools/approval.ts:255-272](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/approval.ts#L255-L272)
- Commands run on the host as the user with no sandbox. (ASI05, T11, LLM05; C4). Evidence: [packages/coding-agent/src/exec/bash-executor.ts:442](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/exec/bash-executor.ts#L442); [packages/utils/src/procmgr.ts:43-55](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/utils/src/procmgr.ts#L43-L55)
- A hijacked session can exfiltrate data and take irreversible actions with no human involved. (ASI01, T6, LLM01; C5). Evidence: [packages/coding-agent/src/tools/settings.ts:312-315](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/settings.ts#L312-L315); [packages/coding-agent/src/tools/settings.ts:700-703](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/settings.ts#L700-L703)
- Repository settings, hooks, extensions and MCP servers load without a workspace-trust decision. (ASI06, ASI04, T1; C6). Evidence: [packages/coding-agent/src/extensibility/extensions/types.ts:521-523](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/extensibility/extensions/types.ts#L521-L523); [packages/coding-agent/src/config/settings.ts:2378-2381](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/config/settings.ts#L2378-L2381)
- Workspace extension modules are imported into the agent process and MCP servers launched without consent. (ASI04, T17, LLM03; C7). Evidence: [packages/coding-agent/src/extensibility/custom-tools/loader.ts:78](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/extensibility/custom-tools/loader.ts#L78); [packages/coding-agent/src/mcp/settings.ts:9-12](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/mcp/settings.ts#L9-L12)

## Criterion details

### C1 Identity & least privilege: 0.00 (high confidence)

omp runs as the user who launches it and uses that user's ambient authority: every shell command and every MCP server it starts receives the agent's full process environment, including any API keys and cloud or git credentials in it. The only scrubbing removes a narrow set of variables such as git repository-location settings, not credentials. There is no per-tool identity or authorization check in code. A hijacked session therefore acts with everything the user can do.

- **S L0:** Tools act with the operator's ambient OS identity and inherited environment; no scoped identity or authorization layer exists. Evidence: [packages/utils/src/procmgr.ts:43-55](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/utils/src/procmgr.ts#L43-L55); [packages/utils/src/env.ts:236-238](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/utils/src/env.ts#L236-L238) (verified)
  - *To reach the next level:* Pass tools a narrowed identity or a deterministic per-request authorization check before credentials are used.
- **C L0:** The shell spawn environment and the MCP stdio launch environment both start from the full process environment. Evidence: [packages/utils/src/procmgr.ts:43-55](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/utils/src/procmgr.ts#L43-L55); [packages/coding-agent/src/mcp/transports/stdio.ts:581-584](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/mcp/transports/stdio.ts#L581-L584) (verified)
  - *To reach the next level:* Give every subprocess a scrubbed environment so at least the main tool path avoids ambient credentials.
- **D L0:** The default install hands the full environment to tools with no least-privilege default. Evidence: [packages/utils/src/procmgr.ts:43-55](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/utils/src/procmgr.ts#L43-L55) (verified)
  - *To reach the next level:* Ship a minimal default environment for tools, widened only by explicit operator configuration.
- **B L0:** Anything the user's environment and home directory can reach (cloud CLIs, git push credentials, provider keys) is reachable from a hijacked shell. Evidence: [packages/utils/src/procmgr.ts:43-55](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/utils/src/procmgr.ts#L43-L55); [packages/coding-agent/src/mcp/transports/stdio.ts:581-584](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/mcp/transports/stdio.ts#L581-L584) (verified)
  - *To reach the next level:* Limit what tools can reach to one project with non-destructive credentials.
- **Cap:** none

### C2 Approval gates: 0.25 (high confidence)

omp has a real approval system: tools declare read, write or exec tiers, prompts show the command, and users can write ordered allow/prompt/deny rules for bash that are matched per segment of compound commands. But it ships with approval mode 'yolo', which auto-approves every tier, so shell commands, file writes, code evaluation and sub-agents run without asking. Sub-agents are always run in yolo mode, and checkpoints are off by default, so most actions cannot be rolled back. Switching tools.approvalMode to always-ask turns the gate on.

- **S L2:** Per-call approval with read/write/exec tiers and user allow/prompt/deny rules; the prompt shows the command but truncates long arguments. Evidence: [packages/coding-agent/src/tools/approval.ts:120-126](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/approval.ts#L120-L126); [packages/coding-agent/src/tools/bash.ts:589-593](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/bash.ts#L589-L593) (verified)
  - *To reach the next level:* Show the complete, untruncated call to the approver for every gated action.
- **C L2:** In the non-yolo modes built-in and MCP tools pass the tier check, but sub-agents are forced to yolo and run their tool calls ungated. Evidence: [packages/coding-agent/src/mcp/tool-bridge.ts:661](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/mcp/tool-bridge.ts#L661); [packages/coding-agent/src/task/executor.ts:1055-1058](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/task/executor.ts#L1055-L1058) (verified)
  - *To reach the next level:* Route sub-agent tool calls through the same approval gate as the parent session.
- **D L0:** The shipped default approval mode is yolo, under which every tier is allowed without a prompt. Evidence: [packages/coding-agent/src/tools/settings.ts:312-315](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/settings.ts#L312-L315); [packages/coding-agent/src/tools/approval.ts:255-272](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/approval.ts#L255-L272) (verified)
  - *To reach the next level:* Turn approval on by default (always-ask or write) and require an explicit flag for yolo.
- **B L1:** Shell, git and network actions are irreversible and checkpoints are off by default. Evidence: [packages/coding-agent/src/tools/settings.ts:687-690](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/settings.ts#L687-L690) (verified)
  - *To reach the next level:* Enable checkpoints or rollback by default for file and code state and preview external actions.
- **Cap:** C2-POWERBYPASS: In the default yolo mode the bash tool, the most powerful action path, runs without approval.

### C3 Tool & action scoping: 0.20 (high confidence)

The default tool set includes general-purpose tools: a raw shell, code evaluation, file write and edit, a URL fetcher and a browser. File paths are resolved but not confined to the workspace, and the fetch tool has no host or internal-address restrictions. A built-in list of catastrophic shell patterns only raises an approval hint, and user deny rules are empty by default. Individual tools can be turned off in settings, but out of the box a misused tool can reach the whole machine and network.

- **S L1:** Validation is limited to a denylist of catastrophic shell patterns and scheme-level read-only checks for internal URLs; shell and fetch take raw input. Evidence: [packages/coding-agent/src/tools/bash.ts:188-208](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/bash.ts#L188-L208); [packages/coding-agent/src/internal-urls/router.ts:403-407](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/internal-urls/router.ts#L403-L407); searched `rg -n -S '169\.254|isPrivateIp|ssrf|blockedHosts|allowedHosts'` in `packages/coding-agent/src/tools packages/coding-agent/src/web` → 0 hits (No internal-address or host allowlist check in the fetch, browser or web tools.) (verified)
  - *To reach the next level:* Validate arguments against allowlists in code: resolved-path containment and host allowlists that block internal addresses.
- **C L1:** Only a few tools (internal URL schemes, bash pattern rules) validate their arguments. Evidence: [packages/coding-agent/src/internal-urls/router.ts:403-407](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/internal-urls/router.ts#L403-L407) (verified)
  - *To reach the next level:* Apply argument validation to most built-in tools.
- **D L1:** Shell, eval, write and sub-agent tools are on by default; tools such as bash can be disabled individually. Evidence: [packages/coding-agent/src/tools/essential-tools.ts:23-37](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/essential-tools.ts#L23-L37); [packages/coding-agent/src/exec/settings.ts:82-85](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/exec/settings.ts#L82-L85) (verified)
  - *To reach the next level:* Offer tool groups and a default set without exec.
- **B L0:** The shell tool can run any command against any host or file the user can reach. Evidence: [packages/coding-agent/src/tools/bash.ts:587](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/bash.ts#L587) (verified)
  - *To reach the next level:* Scope tools to the project workspace.
- **Cap:** none

### C4 Code-execution isolation: 0.00 (high confidence)

Shell commands, code evaluation and MCP servers run directly on the host as the user, with the agent's environment. No sandbox, container or OS-level confinement exists for any execution path; the copy-on-write worktree isolation used for sub-agent tasks separates file changes, not privileges. If model-chosen code is malicious, it gets everything the user has.

- **S L0:** Commands run as same-user subprocesses with no isolation primitive. Evidence: searched `rg -n -S 'landlock|seatbelt|sandbox-exec|bwrap|bubblewrap|seccomp|firejail|nsjail|gvisor|firecracker'` in `packages/coding-agent/src packages/utils/src crates/pi-shell/src` → 8 hits (None is an isolation layer: 4 hits are the SecCompany type in a SEC scraper, the rest are comments about host seccomp/Seatbelt affecting signals, pidfd and write probes.); [packages/coding-agent/src/exec/bash-executor.ts:442](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/exec/bash-executor.ts#L442) (verified)
  - *To reach the next level:* Run commands in an OS sandbox or container with dropped privileges.
- **C L0:** No execution path is sandboxed. Evidence: searched `rg -n -S 'landlock|seatbelt|sandbox-exec|bwrap|bubblewrap|seccomp|firejail|nsjail|gvisor|firecracker'` in `packages/coding-agent/src packages/utils/src crates/pi-shell/src` → 8 hits (None is an isolation layer: 4 hits are the SecCompany type in a SEC scraper, the rest are comments about host seccomp/Seatbelt affecting signals, pidfd and write probes.) (verified)
  - *To reach the next level:* Sandbox at least the main shell and eval tools.
- **D L0:** No sandbox exists to enable. Evidence: searched `rg -n -S 'landlock|seatbelt|sandbox-exec|bwrap|bubblewrap|seccomp|firejail|nsjail|gvisor|firecracker'` in `packages/coding-agent/src packages/utils/src crates/pi-shell/src` → 8 hits (None is an isolation layer: 4 hits are the SecCompany type in a SEC scraper, the rest are comments about host seccomp/Seatbelt affecting signals, pidfd and write probes.) (verified)
  - *To reach the next level:* Ship a sandbox on by default.
- **B L0:** Executed code has host-equivalent reach: the user's home directory, network and inherited credentials. Evidence: [packages/utils/src/procmgr.ts:43-55](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/utils/src/procmgr.ts#L43-L55); [packages/coding-agent/src/exec/bash-executor.ts:442](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/exec/bash-executor.ts#L442) (verified)
  - *To reach the next level:* Confine execution to the workspace without secrets or with allowlisted egress.
- **Cap:** none

### C5 Untrusted input blast radius: 0.25 (high confidence)

omp reads web pages, fetched URLs, files, MCP results and MCP server instructions, and all of it enters the model's context with no structural limit on what a hijacked session can then do. The one specific measure wraps page-provided WebMCP content in nonce-delimited 'untrusted' markers, which helps the model but stops nothing. Because tools run unapproved by default, injected instructions can read secrets and send them out or take irreversible actions without a human.

- **S L1:** Only delimiter-style marking of WebMCP page content; no capability is disabled or gated after untrusted content is read. Evidence: [packages/coding-agent/src/tools/browser/webmcp.ts:261-264](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/browser/webmcp.ts#L261-L264) (verified)
  - *To reach the next level:* Force egress and state-changing tools through approval once untrusted content enters the session.
- **C L1:** One source (WebMCP page content) is marked; fetched pages, files, MCP results and MCP server instructions are not. Evidence: [packages/coding-agent/src/mcp/manager.ts:1808-1812](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/mcp/manager.ts#L1808-L1812); [packages/coding-agent/src/tools/browser/webmcp.ts:261-264](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/browser/webmcp.ts#L261-L264) (verified)
  - *To reach the next level:* Mark or contain fetched content, files and MCP results and instructions as well.
- **D L2:** The WebMCP marking is always applied, but it is only a marker. Evidence: [packages/coding-agent/src/tools/browser/webmcp.ts:282](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/browser/webmcp.ts#L282) (verified)
  - *To reach the next level:* Back the marker with an enforced control that the operator cannot silently disable.
- **B L0:** With approval in yolo mode, fetch and browser on, and the full environment in the shell, a hijacked session can both exfiltrate secrets and take irreversible actions unattended. Evidence: [packages/coding-agent/src/tools/settings.ts:312-315](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/settings.ts#L312-L315); [packages/coding-agent/src/tools/settings.ts:700-703](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/settings.ts#L700-L703) (verified)
  - *To reach the next level:* Require a human before exfiltration-capable or irreversible actions in sessions that read untrusted content.
- **Cap:** C5-WORSTCASE: In the default configuration a hijacked session can leak data and take irreversible actions with no human involved.

### C6 Memory, context & configuration integrity: 0.10 (high confidence)

The long-term memory backends and the learn tool are off by default. But omp performs no project-trust gating, which its own source says: a repository's .omp and .claude settings, hooks, extension modules, MCP servers and instruction files such as AGENTS.md are loaded automatically when you start omp inside it. A cloned repository can therefore change security settings, add tools or start servers without any trust decision, and those changes persist for every session in that checkout.

- **S L0:** Repository-controlled settings, hooks, extensions and MCP servers load with no prompt; project settings merge over user settings unfiltered. Evidence: [packages/coding-agent/src/extensibility/extensions/types.ts:521-523](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/extensibility/extensions/types.ts#L521-L523); [packages/coding-agent/src/config/settings.ts:2378-2381](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/config/settings.ts#L2378-L2381) (verified)
  - *To reach the next level:* Require an explicit workspace-trust decision before loading security-relevant project configuration.
- **C L0:** No auto-loaded path (settings, hooks, extensions, MCP, instruction files) is controlled. Evidence: [packages/coding-agent/src/discovery/claude.ts:259-263](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/discovery/claude.ts#L259-L263); [packages/coding-agent/src/discovery/claude.ts:394-395](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/discovery/claude.ts#L394-L395); [packages/coding-agent/src/discovery/agents-md.ts:20-21](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/discovery/agents-md.ts#L20-L21) (verified)
  - *To reach the next level:* Put at least the hook, extension and MCP project paths behind a trust check.
- **D L1:** Memory backends are off by default; there is no namespace isolation to speak of beyond per-project file layout. Evidence: [packages/coding-agent/src/memory-backend/settings.ts:12-17](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/memory-backend/settings.ts#L12-L17) (verified)
  - *To reach the next level:* Enforce per-project namespaces in code for any persisted memory and project configuration.
- **B L1:** Poisoned project configuration persists across the user's sessions in that checkout and can trigger tool use through hooks and extensions. Evidence: [packages/coding-agent/src/discovery/claude.ts:394-395](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/discovery/claude.ts#L394-L395); [packages/coding-agent/src/discovery/claude.ts:259-263](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/discovery/claude.ts#L259-L263) (verified)
  - *To reach the next level:* Limit persisted context to text output or gated actions, or make it easy to inspect and purge.
- **Cap:** C6-REPOCONFIG: Project .omp/.claude settings, hooks, extension modules and .mcp.json servers load from the working directory without any trust decision and can change approval settings and add tools.

### C7 Third-party extensions: 0.07 (high confidence)

omp loads extension modules, custom tools and hooks as code inside its own process, launches MCP servers as subprocesses with the full environment, and installs plugins from a marketplace without hash or signature checks. Project-level MCP configuration is enabled by default and project extension directories are scanned automatically, so a repository can add running code without a consent step. A malicious extension gets everything the agent has.

- **S L1:** Extensions and MCP servers run whatever the configured source provides; no pinning or integrity verification in the plugin installer. Evidence: searched `rg -n -S 'sha256|integrity|checksum|signature'` in `packages/coding-agent/src/extensibility/plugins` → 0 hits (No integrity verification in the plugin installer or marketplace code.); [packages/coding-agent/src/mcp/transports/stdio.ts:581-584](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/mcp/transports/stdio.ts#L581-L584) (verified)
  - *To reach the next level:* Pin extension and MCP server versions.
- **C L0:** No extension type is verified. Evidence: searched `rg -n -S 'sha256|integrity|checksum|signature'` in `packages/coding-agent/src/extensibility/plugins` → 0 hits (No integrity verification for plugins; MCP servers launch from their configured command.) (verified)
  - *To reach the next level:* Verify at least one extension type (plugins or MCP servers).
- **D L0:** Project MCP config is on by default and project extension directories are discovered automatically, so workspace files add extensions silently. Evidence: [packages/coding-agent/src/mcp/settings.ts:9-12](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/mcp/settings.ts#L9-L12); [packages/coding-agent/src/discovery/claude.ts:259-263](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/discovery/claude.ts#L259-L263) (verified)
  - *To reach the next level:* Require explicit consent showing the exact package or command before enabling any extension.
- **B L0:** Extension modules are imported into the agent process; MCP stdio servers get the full process environment. Evidence: [packages/coding-agent/src/extensibility/custom-tools/loader.ts:78](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/extensibility/custom-tools/loader.ts#L78); [packages/coding-agent/src/mcp/transports/stdio.ts:581-584](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/mcp/transports/stdio.ts#L581-L584) (verified)
  - *To reach the next level:* Run extensions in separate processes with a scrubbed environment.
- **Cap:** C7-RCELOAD: By default, extension modules and MCP servers from the workspace are imported or launched without consent.

### C8 Secrets & sensitive-data protection: 0.25 (high confidence)

Stored provider credentials sit in a local SQLite file restricted to the user, and there is no third-party telemetry; OpenTelemetry export only happens when the operator sets OTEL endpoints. A placeholder-based secret obfuscation and outbound credential redaction exist but are off by default. Shell commands and MCP servers inherit the full environment, so any long-lived key in it is reachable by the model.

- **S L1:** Secrets come from the environment and a plaintext credential database with 0600 permissions; outbound redaction is opt-in. Evidence: [packages/ai/src/auth/sqlite-credential-store.ts:545-552](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/ai/src/auth/sqlite-credential-store.ts#L545-L552); [packages/coding-agent/src/secrets/settings.ts:13-16](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/secrets/settings.ts#L13-L16) (verified)
  - *To reach the next level:* Mask secrets in logs and transcripts by default.
- **C L1:** Only the credential store is protected; subprocess environments, transcripts and model-bound messages are not by default. Evidence: [packages/utils/src/procmgr.ts:43-55](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/utils/src/procmgr.ts#L43-L55); [packages/coding-agent/src/secrets/settings.ts:13-16](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/secrets/settings.ts#L13-L16) (verified)
  - *To reach the next level:* Protect logs and transcripts as well as the credential store.
- **D L2:** No third-party telemetry SDK; OTLP export requires operator-set OTEL endpoints; redaction can be enabled but is off. Evidence: searched `rg -n -S 'posthog|@sentry|mixpanel'` in `packages/coding-agent/src packages/ai/src` → 1 hits (The single hit is a model-name example in a comment, not a telemetry SDK.); [packages/coding-agent/src/telemetry-export.ts:109](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/telemetry-export.ts#L109) (verified)
  - *To reach the next level:* Turn redaction on by default.
- **B L0:** Long-lived provider and service keys in the environment are reachable by every subprocess the model starts. Evidence: [packages/utils/src/procmgr.ts:43-55](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/utils/src/procmgr.ts#L43-L55) (verified)
  - *To reach the next level:* Keep long-lived keys out of tool subprocess environments.
- **Cap:** none
- **Notes:** The opt-in secrets obfuscation (placeholders before provider requests) was considered as an alternative mechanism but did not score above the default once G1 applied.

### C9 Audit & traceability: 0.50 (high confidence)

Every session is written as a structured JSONL transcript under the user's agent directory, including tool calls and results, and sub-agents write child transcripts next to the parent. Entries are handed to the OS as they complete, so a crash loses at most in-flight text. The record has no actor attribution, approval log or tamper protection, and the agent's own shell can edit it.

- **S L2:** Structured per-entry JSONL transcript of messages, tool calls and results. Evidence: [packages/coding-agent/src/session/session-manager.ts:765-773](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/session/session-manager.ts#L765-L773) (verified)
  - *To reach the next level:* Add actor attribution (approver, delegation chain) and correlation IDs across sub-agents.
- **C L2:** Built-in tool calls and sub-agent sessions are recorded; recording of approvals and denials was not verified. Evidence: [packages/coding-agent/src/session/session-manager.ts:362-363](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/session/session-manager.ts#L362-L363) (verified)
  - *To reach the next level:* Record approvals and denials alongside every tool call.
- **D L2:** On by default and stored outside the workspace in the agent directory, but writable by the agent's own shell. Evidence: [packages/utils/src/dirs.ts:956-957](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/utils/src/dirs.ts#L956-L957) (verified)
  - *To reach the next level:* Write the record through a component the model cannot control.
- **B L2:** Completed entries are written synchronously on append (not fsync'd). Evidence: [packages/coding-agent/src/session/session-manager.ts:769-772](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/session/session-manager.ts#L769-L772) (verified)
  - *To reach the next level:* Make the record durable per action and replayable.
- **Cap:** none

### C10 Limits & kill switch: 0.25 (high confidence)

Each tool has an enforced timeout (bash defaults to 5 minutes, up to an hour), sub-agents have a request budget that force-stops at 1.5 times its value, a recursion depth of 2 and a concurrency cap, and stopping kills the shell's whole process group. But the main session has no step, cost or wall-clock limit, and the global tool-timeout ceiling and sub-agent wall-clock limit default to unlimited.

- **S L1:** Per-tool timeouts are enforced but the main loop has no step, token or cost cap. Evidence: [packages/coding-agent/src/tools/tool-timeouts.ts:10-12](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/tool-timeouts.ts#L10-L12); searched `rg -n -S 'maxTurns|maxSteps|maxIterations|turnLimit'` in `packages/agent/src packages/coding-agent/src/session/agent-session.ts` → 0 hits (No step or turn cap in the agent loop.) (verified)
  - *To reach the next level:* Add an enforced step cap for the main session alongside the existing timeouts.
- **C L1:** Tool timeouts apply to every call and sub-agents have their own request budget, depth and concurrency caps, but the main loop itself is unbounded. Evidence: [packages/coding-agent/src/task/settings.ts:272-275](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/task/settings.ts#L272-L275); [packages/coding-agent/src/task/settings.ts:337-340](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/task/settings.ts#L337-L340) (verified)
  - *To reach the next level:* Bound the main loop as well as tool calls.
- **D L1:** The global tool-timeout ceiling and sub-agent wall-clock default to unlimited; the main session has no default limit. Evidence: [packages/coding-agent/src/tools/settings.ts:870-873](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/tools/settings.ts#L870-L873); [packages/coding-agent/src/task/settings.ts:291-294](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/packages/coding-agent/src/task/settings.ts#L291-L294) (verified)
  - *To reach the next level:* Ship sensible default run limits.
- **B L1:** A runaway main session can loop and spend indefinitely; stopping does kill the shell process group. Evidence: [crates/pi-shell/src/process.rs:1625-1630](https://github.com/can1357/oh-my-pi/blob/8aaf115b8e5f54e6bddeffa969e43e290a1efa01/crates/pi-shell/src/process.rs#L1625-L1630) (verified)
  - *To reach the next level:* Add tight per-run time and cost ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Fetched URLs, browser pages, files and MCP results and instructions (packages/coding-agent/src/tools/settings.ts:700-703, packages/coding-agent/src/mcp/manager.ts:1808-1812) · [B] sensitive data/systems: Full user environment and home directory reachable from the shell (packages/utils/src/procmgr.ts:43-55) · [C] state change / egress: Unapproved bash, write, fetch and browser in default yolo mode (packages/coding-agent/src/tools/settings.ts:312-315) · Same default session? Yes

## Highest-impact improvements
1. Default tools.approvalMode to always-ask (or write) and require an explicit flag for yolo. (C2 D L0→L3, +0.150 before caps; Playbook 5)
2. Add a workspace-trust prompt before loading project settings, hooks, extensions and MCP servers. (C6 S L0→L3, +0.225 before caps; Playbook 2)
3. Require consent showing the exact module or command before importing extensions or launching MCP servers, and pin versions. (C7 D L0→L3, +0.150 before caps; Playbook 3)
4. Pass a scrubbed environment to shell commands and MCP servers instead of the full process environment. (C1 C L0→L1, +0.075 before caps; Playbook 4)
5. Add an enforced step cap for the main session. (C10 S L1→L2, +0.075 before caps; Playbook 3 step 3)

## Re-audit log
- C2 S: L3 → L2. Approval prompts truncate long commands (tools/approval.ts:126, tools/bash.ts:589-593), so the approver does not always see the exact call.
- C5 S: L0 → L1. Defending L0: found nonce-delimited untrusted markers for WebMCP page content (tools/browser/webmcp.ts:261-264); credited as delimiting only.
- C2 C: L3 → L2. Sub-agents are forced to yolo mode (task/executor.ts:1055-1058), so their tool calls skip the gate.
- C10 C: L2 → L1. The main session loop has no step, cost or time limit (search over packages/agent/src found no turn cap); only tool calls and sub-agents are bounded.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The project's README and docs/ files were not read; every rating rests on source code alone.
- Scored the interactive CLI; RPC/ACP protocol hosts use different defaults for some settings (memory, task limits) and were not scored separately.
- The Rust native crates, the collab web app, the Python robomp package and the browser relay extension were reviewed only where the TypeScript tools call into them.
- Recording of approval decisions in transcripts was not verified.
