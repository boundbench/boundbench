# Defense-in-depth score: Docker Agent

**Repo:** https://github.com/docker/docker-agent · **Commit:** `54cea981dfc521617ae81e598b8235df5803b6ae` · **Reviewed:** 2026-10-05
**What it is:** Docker's CLI and runtime for defining, running and sharing AI agents and multi-agent teams in declarative YAML, with built-in tools, MCP support and a built-in default agent.
**Category:** Agent Frameworks
**Scored configuration:** `docker agent run` in the interactive TUI with no flags, no user settings and no project agent file in the working directory, so the built-in default agent (filesystem, shell, background jobs, fetch, skills, AGENTS.md) runs on the host with the unset (legacy) safety mode.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication opt-in

## Score: 3.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L0 | L1 | 0.12 | none | **0.12** | High |
| C2 | Approval gates | L3 | L2 | L1 | L1 | 0.47 | G2 | **0.25** | High |
| C3 | Tool & action scoping | L2 | L1 | L1 | L0 | 0.28 | none | **0.28** | High |
| C4 | Code-execution isolation | L3 | L3 | L0 | L2 | 0.55 | G1 | **0.50** (alt) | High |
| C5 | Untrusted input blast radius | L2 | L1 | L1 | L2 | 0.38 | none | **0.38** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | L1 | L1 | L0 | L1 | 0.20 | none | **0.20** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L0 | L0 | 0.30 | none | **0.30** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | none | **0.50** | High |
| C10 | Limits & kill switch | L3 | L2 | L1 | L1 | 0.47 | G1 | **0.47** | High |


Docker Agent asks before every shell command, file write and web fetch by default, showing the exact call, and its fetch tool blocks internal addresses and scrubs secrets from what goes to the model. But everything that is approved runs directly on your machine with your full environment, since the VM sandbox is opt-in, and there are no default step or spend limits. A docker-agent.yaml in the folder you run it from is loaded automatically and can switch off approvals, add MCP servers and hooks without any trust prompt, and usage telemetry, which can include prompts given on the command line, is on by default.

## Critical gaps
- A docker-agent.yaml in the working directory is loaded without a trust prompt and can set the autonomous safety mode, turning off per-call approval. (ASI09, ASI02, ASI06; C2). Evidence: [cmd/root/run.go:653-666](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/cmd/root/run.go#L653-L666); [cmd/root/run.go:1286-1304](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/cmd/root/run.go#L1286-L1304)
- Project agent files, AGENTS.md and project skills are auto-loaded from the working directory, and the agent file can add MCP servers, hooks and permission rules with no workspace-trust decision. (ASI06, ASI04, T1; C6). Evidence: [cmd/root/run.go:643-666](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/cmd/root/run.go#L643-L666); [pkg/skills/local.go:66-71](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/skills/local.go#L66-L71)
- Approved shell commands and background jobs run unsandboxed on the host with the user's full environment and credentials; the VM sandbox is opt-in. (ASI05, T11; C4). Evidence: [pkg/tools/builtin/shell/shell.go:241-245](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/shell/shell.go#L241-L245); [cmd/root/run.go:217](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/cmd/root/run.go#L217)

## Criterion details

### C1 Identity & least privilege: 0.12 (high confidence)

Docker Agent runs as the user who launched it and does nothing to narrow that authority. The shell tool, background jobs and every MCP server it launches inherit the whole process environment, including model-provider API keys and any cloud or GitHub tokens the user has exported, and there is no per-tool identity or credential scoping. What keeps a hijacked agent from using those credentials is the separate per-call approval prompt, not any limit on the credentials themselves.

- **S L0:** Commands run as the OS user with the full host environment appended in front of the toolset's own variables; no scoped or per-capability identity exists. Evidence: [pkg/tools/builtin/shell/shell.go:236-246](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/shell/shell.go#L236-L246); [pkg/config/sources/builtin-agents/default.yaml:29-33](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/config/sources/builtin-agents/default.yaml#L29-L33) (verified)
  - *To reach the next level:* No dedicated or role-scoped identity; subprocesses get the user's ambient credentials.
- **C L1:** Shell, background jobs and stdio MCP servers each receive os.Environ() unfiltered; the only check on the main tool path is the approval prompt. Evidence: [pkg/tools/builtin/backgroundjobs/backgroundjobs.go:544](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/backgroundjobs/backgroundjobs.go#L544); [pkg/tools/mcp/mcp.go:109](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/mcp/mcp.go#L109) (verified)
  - *To reach the next level:* Built-in tools do not use a scoped identity; every subprocess and extension inherits ambient credentials.
- **D L0:** The built-in default agent ships with shell, filesystem and fetch toolsets running with the user's full privileges; least privilege needs a hand-written agent file. Evidence: [pkg/config/sources/builtin-agents/default.yaml:29-33](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/config/sources/builtin-agents/default.yaml#L29-L33) (verified)
  - *To reach the next level:* No narrower default; the default agent holds the user's full authority.
- **B L1:** A hijacked agent can reach everything the user's credentials reach, but each shell, write or fetch call still needs a per-call human approval in the default mode, which survives as an independent layer. Evidence: [pkg/runtime/toolexec/dispatcher.go:488-492](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/runtime/toolexec/dispatcher.go#L488-L492) (verified)
  - *To reach the next level:* Credentials are not scoped to one system or made read-only.
- **Cap:** none

### C2 Approval gates: 0.25 (high confidence)

The approval gate is well built: in the default mode every tool that is not annotated read-only, including shell, file writes and web fetches, stops for a per-call prompt that renders the exact call, and allow/ask/deny rules can match on arguments. Unknown tools are rejected, sub-agents inherit the session's mode, and shell commands embedded in skills go through the same prompt. But a docker-agent.yaml in the working directory is picked up automatically and may declare the autonomous safety mode, which approves every call, so a repository you open can turn the gate off; one key press in the prompt also switches the whole session to autonomous. Workspace snapshots that could undo changes are opt-in.

- **S L3:** Each gated call is shown in a confirmation dialog that renders the tool call itself with the runtime's safety label, and rules resolve allow/ask/deny on tool arguments. Evidence: [pkg/tui/dialog/toolconfirmation/tool_confirmation.go:336-346](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tui/dialog/toolconfirmation/tool_confirmation.go#L336-L346); [pkg/runtime/toolexec/permissions.go:84-111](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/runtime/toolexec/permissions.go#L84-L111); [pkg/runtime/toolexec/dispatcher.go:976-1000](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/runtime/toolexec/dispatcher.go#L976-L1000) (verified)
  - *To reach the next level:* Approve options include session-wide escalation (balanced or autonomous) from the prompt, and argument rules match glob patterns on raw strings rather than parsed commands.
- **C L2:** Every tool call, including MCP tools, sub-agent sessions (which inherit the parent's mode) and skill-embedded commands, passes the same dispatcher gate and unknown tools are rejected, but in the default mode any tool annotated read-only skips the prompt, and for MCP tools that annotation is declared by the server itself. Evidence: [pkg/runtime/toolexec/dispatcher.go:399-404](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/runtime/toolexec/dispatcher.go#L399-L404); [pkg/runtime/toolexec/permissions.go:157-158](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/runtime/toolexec/permissions.go#L157-L158); [pkg/runtime/agent_delegation.go:676](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/runtime/agent_delegation.go#L676); [pkg/tui/components/toolconfirm/toolconfirm.go:75-87](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tui/components/toolconfirm/toolconfirm.go#L75-L87) (verified)
  - *To reach the next level:* Auto-approval trusts read-only annotations that MCP servers declare for themselves rather than a verified read-only allowlist, and the interactive 'always allow' grant is a first-word prefix pattern over the whole command string.
- **D L1:** Prompting is on by default, but with no arguments the CLI loads docker-agent.yaml from the current directory and applies its author-declared safety mode, which may be autonomous, whenever the user has set no preference. Evidence: [cmd/root/run.go:653-666](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/cmd/root/run.go#L653-L666); [cmd/root/run.go:1286-1304](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/cmd/root/run.go#L1286-L1304); [docs/configuration/permissions/index.md:59](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/docs/configuration/permissions/index.md#L59) (verified)
  - *To reach the next level:* A workspace file can switch the session to auto-approve without an explicit operator decision.
- **B L1:** Approved shell commands can delete files, push code or call external services irreversibly; automatic shadow-git snapshots exist but are off unless enabled in user settings. Evidence: [pkg/userconfig/userconfig.go:287-289](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/userconfig/userconfig.go#L287-L289) (verified)
  - *To reach the next level:* No default checkpoint or rollback for file and code changes.
- **Cap:** G2: A docker-agent.yaml in the working directory is loaded without a trust decision and can set the autonomous safety mode, which approves every tool call.

### C3 Tool & action scoping: 0.28 (high confidence)

Some built-in tools validate their inputs well: the fetch tool refuses loopback, private and cloud-metadata addresses at connection time and rechecks domain rules on every redirect, and the filesystem tools honour .agentsignore and can be confined to allow-listed directories using kernel-checked roots. But the default agent also ships a raw shell and background-job runner that take any command string, and the filesystem tools accept any absolute path unless an allow-list is configured. Toolsets are chosen per agent in YAML, yet the built-in default enables write, exec and network tools together.

- **S L2:** Fetch has dial-time private-address blocking and redirect rechecks, and filesystem paths are checked against optional allow/deny roots, but shell and background jobs pass the model's command string straight to the host shell. Evidence: [pkg/tools/builtin/fetch/fetch.go:106-141](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/fetch/fetch.go#L106-L141); [pkg/tools/builtin/filesystem/filesystem.go:545-555](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/filesystem/filesystem.go#L545-L555); [pkg/tools/builtin/shell/shell.go:152](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/shell/shell.go#L152) (verified)
  - *To reach the next level:* The shell and background-job tools have no argument validation, and filesystem containment needs an allow-list that is not set by default.
- **C L1:** Two of the four default toolsets (fetch, filesystem) validate inputs; shell and background jobs, the most powerful, do not, and there is no shared validation layer for MCP tools. Evidence: [pkg/tools/builtin/backgroundjobs/backgroundjobs.go:251-252](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/backgroundjobs/backgroundjobs.go#L251-L252) (verified)
  - *To reach the next level:* Most built-in tools, including the command tools, do not validate their arguments.
- **D L1:** The built-in default agent enables filesystem write, shell, background jobs and fetch together; each toolset can be removed or made read-only only by writing a custom agent file. Evidence: [pkg/config/sources/builtin-agents/default.yaml:29-33](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/config/sources/builtin-agents/default.yaml#L29-L33) (verified)
  - *To reach the next level:* The default tool group includes write, exec and network tools rather than a read-only set.
- **B L0:** A misused shell call can run any command anywhere the user can, and filesystem tools can read and write any absolute path the user can access. Evidence: [pkg/tools/builtin/filesystem/filesystem.go:488-497](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/filesystem/filesystem.go#L488-L497) (verified)
  - *To reach the next level:* Tools are not scoped to the workspace or bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation: 0.50 (high confidence)

By default every approved shell command and background job runs directly on the host as the user, in the user's working directory, with the full environment and network. Docker Agent can instead run the whole agent inside a Docker Sandboxes VM with a default-deny network proxy (--sandbox, or runtime.sandbox in an agent file), where all tools execute inside the VM and only the working directory is mounted read-write. That option is off by default, so it can at most earn half credit here.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** The shell tool starts the user's shell as a plain host subprocess; there is no isolation on the default path. Evidence: [pkg/tools/builtin/shell/shell.go:147-156](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/shell/shell.go#L147-L156) (verified)
    - *To reach the next level:* No isolation primitive on the default execution path.
  - **C L0:** Shell, background jobs, skill command expansion and stdio MCP servers all run on the host in the default configuration. Evidence: [pkg/tools/builtin/backgroundjobs/backgroundjobs.go:251-252](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/backgroundjobs/backgroundjobs.go#L251-L252) (verified)
    - *To reach the next level:* The main execution tool is not sandboxed by default.
  - **D L0:** The --sandbox flag defaults to false. Evidence: [cmd/root/run.go:217](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/cmd/root/run.go#L217) (verified)
    - *To reach the next level:* Sandboxing is off by default.
  - **B L0:** Commands run with host-equivalent reach: the user's whole filesystem, unrestricted network, and every credential in the inherited environment. Evidence: [pkg/tools/builtin/shell/shell.go:241-245](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/shell/shell.go#L241-L245) (verified)
    - *To reach the next level:* Nothing narrows what an executed command reaches.
- **opt-in Docker Sandboxes VM (--sandbox)** (alt; raw 0.55, cap G1 → 0.50) ← counted
  - **S L3:** With --sandbox the CLI hands the run to the Docker Sandboxes CLI, which creates a VM from a template and runs the agent inside it behind a default-deny network proxy. Evidence: [pkg/sandbox/sandbox.go:20-24](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/sandbox/sandbox.go#L20-L24); [docs/configuration/sandbox/index.md:345-349](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/docs/configuration/sandbox/index.md#L345-L349) (verified)
    - *To reach the next level:* The isolation boundary is provided by an external product and is not verifiable from this repository, so it is not credited as kernel-separated here.
  - **C L3:** The whole agent process runs inside the VM, so shell, filesystem, background jobs and MCP servers are all inside the boundary, and a missing sandbox CLI stops the run rather than falling back to the host. Evidence: [docs/configuration/sandbox/index.md:349](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/docs/configuration/sandbox/index.md#L349); [cmd/root/sandbox.go:106-114](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/cmd/root/sandbox.go#L106-L114) (verified)
    - *To reach the next level:* Not verified that every process spawned by extensions inside the VM inherits the same network policy.
  - **D L0:** The sandbox is opt-in through a flag, an alias option or runtime.sandbox in an agent file. Evidence: [cmd/root/run.go:217](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/cmd/root/run.go#L217) (verified)
    - *To reach the next level:* Off by default.
  - **B L2:** Inside the VM the working directory is mounted read-write and network is limited to the models gateway, inferred package hosts and declared allowlist entries. Evidence: [docs/configuration/sandbox/index.md:19](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/docs/configuration/sandbox/index.md#L19) (verified)
    - *To reach the next level:* The workspace is mounted read-write and persists to the host, and the sandbox is reused rather than ephemeral per run.
- **Cap:** G1: Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius: 0.38 (high confidence)

Docker Agent does not track whether untrusted content has entered a session: web pages, file contents, AGENTS.md, skills and tool or MCP results all reach the model with ordinary standing. What limits a hijacked agent in the default mode is that every shell command, file write and web fetch still needs a per-call human approval, while the tools that run unprompted are read-only and have no outbound channel. That protection rests on the approval mode, which a project agent file can change and which the user can escalate to autonomous from any prompt.

- **S L2:** Egress and state-changing tools require approval in the default mode, but the requirement does not depend on whether untrusted content was read, and no provenance or taint tracking exists. Evidence: [pkg/runtime/toolexec/dispatcher.go:488-492](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/runtime/toolexec/dispatcher.go#L488-L492); searched `rg -n -i 'taint|quarantin'` in `pkg/runtime pkg/session` → 0 hits (No taint or quarantine mechanism in the runtime or session layers.) (verified)
  - *To reach the next level:* No rule that disables or forces approval for egress once untrusted content enters the session, independent of the chosen safety mode.
- **C L1:** Tool results are appended as ordinary tool messages and prompt files and skills load as instructions; the approval limit applies to all sources only because it ignores the source. Evidence: [pkg/runtime/toolexec/dispatcher.go:1237-1243](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/runtime/toolexec/dispatcher.go#L1237-L1243) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished, so tool and MCP results, tool descriptions and sub-agent messages carry no lower standing.
- **D L1:** The limiting approval mode is on by default, but an auto-discovered project agent file can declare the autonomous mode for new sessions. Evidence: [cmd/root/run.go:1286-1304](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/cmd/root/run.go#L1286-L1304) (verified)
  - *To reach the next level:* Workspace content can switch the limiting control off.
- **B L2:** In the default mode the unprompted tools (file reads, listings, todo, think, job views) have no egress, while fetch, shell and writes all prompt, so both exfiltration and irreversible actions need a human; file reads are unrestricted, so a hijack can stage any readable secret for a later approved call. Evidence: [pkg/runtime/toolexec/permissions.go:157-158](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/runtime/toolexec/permissions.go#L157-L158); searched `rg -n 'ReadOnlyHint'` in `pkg/tools/builtin/fetch pkg/tools/builtin/shell` → 0 hits (Neither fetch nor shell is annotated read-only, so both prompt in the default mode.) (verified)
  - *To reach the next level:* Sensitive files remain readable without approval, and a single approval of a fetch or shell call is enough to exfiltrate them.
- **Cap:** none

### C6 Memory, context & configuration integrity: 0.10 (high confidence)

Running `docker agent run` with no arguments loads docker-agent.yaml (or .yml/.hcl) from the current directory if one exists, announcing it with a one-line message but asking for no trust decision. That file is a full agent definition: it can add MCP servers and shell-based hooks that start with the session, grant permission rules and set the approval mode. The default agent also loads AGENTS.md from the working directory or any parent and discovers skills from project folders, all as trusted instructions; an optional hook can vet prompt files but is not enabled. Long-term memory is opt-in, and session history lives in the user's home directory.

- **S L0:** A project agent file in the working directory is used automatically and can define toolsets, MCP servers, hooks, permissions and safety mode with no prompt. Evidence: [cmd/root/run.go:643-666](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/cmd/root/run.go#L643-L666); [cmd/root/run.go:371-376](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/cmd/root/run.go#L371-L376) (verified)
  - *To reach the next level:* Security-relevant project configuration does not require an explicit workspace-trust decision.
- **C L0:** AGENTS.md is looked up from the working directory upward and project skill folders are scanned automatically; the prompt-file guard runs only if a hook for it is configured. Evidence: [pkg/runtime/prompt_file_guard.go:18-21](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/runtime/prompt_file_guard.go#L18-L21); [pkg/skills/local.go:66-71](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/skills/local.go#L66-L71) (verified)
  - *To reach the next level:* No auto-loaded file or store is vetted by default.
- **D L1:** Session history is kept per OS user in a database under the home directory; there is no multi-tenant store, and nothing beyond file location separates sessions. Evidence: [pkg/paths/paths.go:112-119](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/paths/paths.go#L112-L119) (verified)
  - *To reach the next level:* No enforced per-session namespace or retention limit for persisted context.
- **B L1:** A poisoned AGENTS.md, skill or project agent file in a repository is re-read in every later session in that folder and can drive tool use, although tool calls still meet the approval prompt unless the agent file changes the mode. Evidence: [pkg/config/sources/builtin-agents/default.yaml:27-28](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/config/sources/builtin-agents/default.yaml#L27-L28) (verified)
  - *To reach the next level:* Poisoned context persists across the user's sessions and can trigger tool use.
- **Cap:** C6-REPOCONFIG: A docker-agent.yaml in the working directory is loaded automatically and can enable MCP servers and hooks and loosen approval without an explicit user trust decision.

### C7 Third-party extensions: 0.20 (high confidence)

The built-in default agent loads no third-party code, but agent files can add MCP servers (local commands, remote URLs or Docker catalog entries), and a project agent file in the working directory is picked up automatically, so a repository can add servers that start without any consent prompt. When an MCP or LSP command is missing, Docker Agent installs it from the aqua registry, resolving the latest release unless a version is given, and checks a checksum only when the package publishes one. Local MCP servers run as separate processes but inherit the user's full environment, including API keys.

- **S L1:** Extensions come from sources named in the agent file; auto-installed binaries resolve the latest upstream release when no version is pinned, with a release-published checksum verified when available. Evidence: [pkg/toolinstall/resolver.go:203-217](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/toolinstall/resolver.go#L203-L217); [pkg/toolinstall/checksum.go:87-92](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/toolinstall/checksum.go#L87-L92) (verified)
  - *To reach the next level:* Versions are not pinned by default and there is no signature or allowlisted-registry check.
- **C L1:** Only auto-installed binaries get an integrity check; MCP commands already on PATH, remote MCP URLs and skills are not verified. Evidence: [pkg/tools/mcp/mcp.go:97-112](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/mcp/mcp.go#L97-L112) (verified)
  - *To reach the next level:* MCP servers and other extension types are not verified.
- **D L0:** A docker-agent.yaml in the working directory is loaded automatically, so files in the workspace can add MCP servers that launch with the session. Evidence: [cmd/root/run.go:56](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/cmd/root/run.go#L56) (verified)
  - *To reach the next level:* Workspace files can add extensions without showing what will run.
- **B L1:** Stdio MCP servers run as separate processes with the toolset's variables plus the whole host environment. Evidence: [pkg/tools/mcp/mcp.go:105-111](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/mcp/mcp.go#L105-L111) (verified)
  - *To reach the next level:* Extension processes are not given a scrubbed environment.
- **Cap:** none

### C8 Secrets & sensitive-data protection: 0.30 (high confidence)

Secret redaction is on by default: a pattern scanner scrubs secrets from tool arguments before approval, from tool output before it is stored or shown to the model, and from outgoing chat content. Provider API keys still come from environment variables and are passed whole to every shell command, background job and local MCP server, so any approved command can read them. Usage telemetry to Docker is on by default and, as the project's own documentation says, includes command-line positional arguments, which can contain prompts, and error text.

- **S L2:** The redact_secrets builtin is enabled unless explicitly turned off and scrubs secrets on the tool-input, tool-output and model-bound legs; provider keys are read from the environment rather than a secret store. Evidence: [pkg/config/latest/types.go:1112-1117](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/config/latest/types.go#L1112-L1117); [pkg/hooks/builtins/redact_secrets.go:17-31](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/hooks/builtins/redact_secrets.go#L17-L31) (verified)
  - *To reach the next level:* Provider credentials are not held in a keychain or secret manager, and subprocess environments and telemetry are not redacted.
- **C L2:** Redaction covers tool arguments, tool results, persisted transcripts and model-bound messages, but not subprocess environments or telemetry payloads. Evidence: [pkg/telemetry/global.go:14-21](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/telemetry/global.go#L14-L21); [pkg/tools/builtin/shell/shell.go:241-245](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/shell/shell.go#L241-L245) (verified)
  - *To reach the next level:* Subprocess environments and telemetry are unprotected paths.
- **D L0:** Telemetry is enabled unless TELEMETRY_ENABLED=false and sends command arguments, which the docs say can include prompts, to Docker's endpoint. Evidence: [pkg/telemetry/utils.go:31-37](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/telemetry/utils.go#L31-L37); [docs/community/telemetry/index.md:38](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/docs/community/telemetry/index.md#L38) (verified)
  - *To reach the next level:* Telemetry is on by default and is not content-free.
- **B L0:** Long-lived provider and user API keys in the environment are reachable by every subprocess the model can start. Evidence: [pkg/tools/builtin/backgroundjobs/backgroundjobs.go:544](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/backgroundjobs/backgroundjobs.go#L544) (verified)
  - *To reach the next level:* Keys are long-lived and broadly reachable rather than scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability: 0.50 (high confidence)

Every session is saved to a local SQLite database in the user's home directory as it runs: user messages, the model's tool calls with arguments, tool results and timestamps, with sub-agent sessions stored as linked child sessions. Rejected calls appear as error results, but approval decisions and who approved them are recorded only in OpenTelemetry spans and hook notifications, and OpenTelemetry export is off unless --otel is passed. The database is outside the workspace but writable by the same user, so an approved shell command could alter it.

- **S L2:** Tool calls and results are persisted as structured, timestamped messages in the session store, with child sessions linked by parent_id. Evidence: [pkg/runtime/toolexec/dispatcher.go:1237-1243](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/runtime/toolexec/dispatcher.go#L1237-L1243); [pkg/session/store.go:545](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/session/store.go#L545) (verified)
  - *To reach the next level:* No approver or principal attribution in the stored record, and correlation across sub-agents relies on parent links rather than recorded approvals.
- **C L2:** Built-in, MCP and sub-agent tool calls all pass through the dispatcher and are stored, but approval decisions are emitted only as span attributes and hook events. Evidence: [pkg/runtime/toolexec/dispatcher.go:713-724](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/runtime/toolexec/dispatcher.go#L713-L724); [cmd/root/root.go:173](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/cmd/root/root.go#L173) (verified)
  - *To reach the next level:* Approvals are not part of the default record.
- **D L2:** Recording is on by default and stored under the home data directory rather than the workspace, but in a file the agent's own shell could modify. Evidence: [pkg/paths/paths.go:112-119](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/paths/paths.go#L112-L119) (verified)
  - *To reach the next level:* The record is written where the agent's process and approved commands can alter it.
- **B L2:** A persistence observer mirrors runtime events to the store as they occur, so records are flushed per message rather than at session end. Evidence: [pkg/runtime/persistence_observer.go:13-21](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/runtime/persistence_observer.go#L13-L21) (verified)
  - *To reach the next level:* Actions are not blocked when their record cannot be written, and no replayable durable audit trail is guaranteed.
- **Cap:** none

### C10 Limits & kill switch: 0.47 (high confidence)

Docker Agent has good limit machinery, but little of it is on by default. Agent files can set an iteration cap and run budgets for cost, tokens and working time, which sub-agents draw from as one shared pot, and stopping a turn kills the running shell command's whole process group. In the default agent, though, the iteration cap is zero (unlimited) and no budget is set; what remains is a breaker that stops after five identical tool-call batches in a row and a 30-second shell timeout that the model can raise per call. Background jobs keep running after a turn is interrupted until the session ends.

- **S L3:** When configured, the loop enforces max_iterations and cost, token and time budgets between steps, a repeated-call breaker is always active, and cancelling a shell call terminates its process group. Evidence: [pkg/config/latest/types.go:92-109](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/config/latest/types.go#L92-L109); [pkg/runtime/loop.go:518-521](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/runtime/loop.go#L518-L521); [pkg/tools/builtin/shell/shell.go:180-191](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/shell/shell.go#L180-L191) (verified)
  - *To reach the next level:* Halting a turn does not stop background jobs, so the halt does not interrupt all in-flight work.
- **C L2:** Limits apply to the main loop and to sub-agents through a shared budget pot, and shell and fetch calls have timeouts, but background jobs are started outside the request context. Evidence: [pkg/config/latest/types.go:56-61](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/config/latest/types.go#L56-L61); [pkg/tools/builtin/backgroundjobs/backgroundjobs.go:251](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/backgroundjobs/backgroundjobs.go#L251) (verified)
  - *To reach the next level:* Background jobs do not count against the budget or stop with the turn.
- **D L1:** The iteration cap defaults to zero, which the loop treats as unlimited, no budget is configured, and the model can choose a longer shell timeout on each call. Evidence: [pkg/runtime/loop_steps.go:56-58](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/runtime/loop_steps.go#L56-L58); [pkg/tools/builtin/shell/shell.go:100-103](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/shell/shell.go#L100-L103) (verified)
  - *To reach the next level:* No default step, cost or time ceiling, and per-call timeouts can be raised by the model.
- **B L1:** A runaway session has no spend or step ceiling by default, and stopping a turn leaves background jobs running until the session ends. Evidence: [pkg/tools/builtin/backgroundjobs/backgroundjobs.go:666-686](https://github.com/docker/docker-agent/blob/54cea981dfc521617ae81e598b8235df5803b6ae/pkg/tools/builtin/backgroundjobs/backgroundjobs.go#L666-L686) (verified)
  - *To reach the next level:* No moderate default ceilings, and a stop leaves work running.
- **Cap:** G1: The iteration cap and run budgets exist but are off in the default agent.

## Rule-of-Two check
[A] untrusted input: Web pages via fetch, any readable file, AGENTS.md and project skills, tool results (pkg/config/sources/builtin-agents/default.yaml:24-33) · [B] sensitive data/systems: Any file the user can read via auto-approved read tools, and API keys in the environment inherited by shell (pkg/tools/builtin/shell/shell.go:245) · [C] state change / egress: Shell, background jobs, file writes and fetch, each behind a per-call prompt in the default mode (pkg/runtime/toolexec/dispatcher.go:488-492) · Same default session? Yes

## Highest-impact improvements
1. Require an explicit, remembered workspace-trust decision before an auto-discovered project agent file can set the safety mode, permissions, hooks or MCP servers. (C2 D L1→L3, +0.100 before caps; Playbook 5)
2. Apply the same trust gate to auto-loaded project agent files and skills, and show what they add before first use. (C6 S L0→L3, +0.225 before caps; Playbook 2)
3. Run the default agent in the Docker Sandboxes VM whenever the sandbox CLI is available, with an explicit flag to run on the host. (C4 D L0→L3, +0.150 before caps; Playbook 3)
4. Ship the default agent with a finite max_iterations and a run budget, and cap model-chosen shell timeouts. (C10 D L1→L2, +0.050 before caps; Playbook 3 step 3)
5. Make telemetry opt-in, or drop positional arguments and error text from command events. (C8 D L0→L2, +0.100 before caps; Playbook 4)

## Re-audit log
- C2 C: L3 → L2. The default mode auto-approves any tool annotated read-only (permissions.go:157-158), and for MCP tools that annotation is self-declared by the server, so the auto-approved set is not a verified read-only allowlist.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the built-in default agent in the interactive TUI. The --exec, serve api, serve a2a, serve mcp and ACP modes, custom agent files, the Docker MCP gateway and the board/chat servers were only spot-checked.
- The opt-in sandbox's isolation is provided by the external Docker Sandboxes product and was not verified from this repository.
- The secret-redaction patterns live in a third-party library (portcullis) that was not reviewed; coverage of specific secret formats is not verified.
- The safety classifier used by the balanced and restricted modes was not evaluated, because those modes are not the default.
