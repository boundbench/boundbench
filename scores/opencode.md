# Defense-in-Depth Score: OpenCode

**Repo:** https://github.com/anomalyco/opencode · **Commit:** `907b3bc518fa48e90e8ec24dd327d13eee71c36c` · **Reviewed:** 2026-10-03
**What it is:** The open source coding agent (TUI/desktop)
**Category:** Coding
**Scored configuration:** Interactive `opencode` TUI from packages/opencode with no flags, default build agent, fresh install, opened in a git repository.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 1.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L2 | L2 | L0 | L2 | 0.40 | C2-POWERBYPASS | **0.25** | High |
| C3 | Tool & action scoping | L1 | L2 | L1 | L0 | 0.28 | — | **0.28** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L0 | 0.05 | C6-REPOCONFIG | **0.05** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | C7-RCELOAD | **0.00** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L0 | 0.35 | — | **0.35** | High |


Out of the box, OpenCode's default agent runs shell commands, edits files and fetches any URL without asking, on the host with no sandbox and with the full environment, so a prompt injection can both leak secrets and take irreversible actions unattended. Worse, opening it in a cloned repository silently loads that repo's opencode.json and .opencode directory, which can import and run custom tools and plugins in-process and launch MCP servers, with no trust prompt. A capable permission engine exists, but you have to configure it to ask.

## Critical gaps
- The agent acts with the user's full ambient authority: every shell command and MCP server receives the entire process environment. (ASI03, T3; C1) — [packages/opencode/src/tool/shell.ts:416-426](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L416-L426); [packages/opencode/src/mcp/index.ts:347-356](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/mcp/index.ts#L347-L356)
- The default build agent allows every tool, so the shell runs without human approval. (ASI02, ASI09; C2) — [packages/opencode/src/agent/agent.ts:119-121](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/agent/agent.ts#L119-L121); [packages/opencode/src/tool/shell.ts:283-290](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L283-L290)
- No sandbox: model-chosen commands run directly on the host as the user, as the maintainers' own security policy states. (ASI05, T11; C4) — [SECURITY.md:15-17](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/SECURITY.md#L15-L17); [packages/opencode/src/tool/shell.ts:303-309](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L303-L309)
- A hijacked session can exfiltrate data via webfetch or shell and take irreversible actions with no human involved. (ASI01, LLM01; C5) — [packages/opencode/src/agent/agent.ts:119-121](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/agent/agent.ts#L119-L121); [packages/opencode/src/tool/webfetch.ts:35-48](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/webfetch.ts#L35-L48)
- A repository's opencode.json and .opencode/ files load without a trust decision and can add tools, plugins, MCP servers and permissions. (ASI06, ASI04; C6) — [packages/opencode/src/config/config.ts:420-424](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/config/config.ts#L420-L424); [packages/opencode/src/config/config.ts:438-479](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/config/config.ts#L438-L479)
- Workspace custom tools/plugins and unpinned npm plugins are imported and executed in-process without consent. (ASI04, T17; C7) — [packages/opencode/src/tool/registry.ts:183-192](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/registry.ts#L183-L192); [packages/opencode/src/plugin/loader.ts:139](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/plugin/loader.ts#L139); [packages/opencode/src/plugin/shared.ts:32](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/plugin/shared.ts#L32)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

OpenCode runs entirely with the developer's own operating-system authority and does nothing to narrow it. Every shell command and every locally launched MCP server receives the full parent environment, so API keys, cloud credentials, SSH agent sockets and gh/git auth are all reachable. There is no per-tool identity or credential scoping, and the permission rules decide which tools run, not which credentials they get. A hijacked session can therefore act as the user across every service the machine is logged into.

- **S L0:** Ambient authority: the shell tool spawns commands with the whole process environment and no scoped identity. — [packages/opencode/src/tool/shell.ts:416-426](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L416-L426) (verified)
  - *To reach the next level:* Give tools a scrubbed environment and narrowed, task-specific credentials instead of the operator's full environment.
- **C L0:** No authorization layer governs credentials on any path; shell and MCP stdio servers both inherit process.env. — [packages/opencode/src/tool/shell.ts:416-426](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L416-L426); [packages/opencode/src/mcp/index.ts:347-356](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/mcp/index.ts#L347-L356) (verified)
  - *To reach the next level:* Route every tool and extension through one credential/authorization layer instead of passing the full environment.
- **D L0:** The default install runs with the user's full privileges and the default build agent allows every tool. — [packages/opencode/src/agent/agent.ts:119-121](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/agent/agent.ts#L119-L121); [packages/opencode/src/tool/shell.ts:416-426](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L416-L426) (verified)
  - *To reach the next level:* Ship a near-minimal default (no ambient secrets in subprocesses, write tools requiring elevation).
- **B L0:** A hijacked agent can use every credential on the machine (cloud CLIs, gh, ssh) through the shell, i.e. the user's entire account across services. — [packages/opencode/src/tool/shell.ts:416-426](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L416-L426); [packages/opencode/src/agent/agent.ts:119-121](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/agent/agent.ts#L119-L121) (verified)
  - *To reach the next level:* Limit what a hijacked session can reach, e.g. by withholding ambient credentials from subprocesses.
- **Cap:** none

### C2 Approval gates — 0.25 (high)

OpenCode has a real permission engine: it can ask before each shell command (parsed per sub-command with tree-sitter) and shows file diffs before edits, with once/always/reject answers. But the default build agent's ruleset is "allow everything", so in a fresh install shell commands, file writes, web fetches and MCP tool calls run without any human approval; only reads outside the project, .env reads and repeated identical calls prompt. The README and SECURITY.md describe the permission system as prompting before commands, which the shipped defaults do not do; enforcement in another mode also does not cover every path. Git-based snapshots let file changes be reverted, but pushes, deletions outside the project and network calls cannot be undone.

- **S L2:** When rules are set to ask, bash prompts show the exact command and edits show the diff, but MCP tool approvals carry empty metadata and 'always' widens to a command-prefix wildcard. — [packages/opencode/src/tool/shell.ts:283-290](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L283-L290); [packages/opencode/src/session/tools.ts:408](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/session/tools.ts#L408); [packages/opencode/src/tool/shell.ts:407-410](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L407-L410) (verified)
  - *To reach the next level:* Show the exact arguments for every tool kind (including MCP) and add argument-level policy so the approved call is exactly what runs.
- **C L2:** Built-in tools and MCP tools call ctx.ask, but custom tools from .opencode/tool and plugins only cross the gate if the plugin itself calls ask. — [packages/opencode/src/tool/registry.ts:143-154](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/registry.ts#L143-L154); [packages/opencode/src/session/tools.ts:402-409](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/session/tools.ts#L402-L409) (verified)
  - *To reach the next level:* Force every tool, including plugin and custom tools, through the gate in the executor rather than relying on each tool to call ask.
- **D L0:** Approval is effectively opt-in: the default ruleset allows every permission, and evaluate() only falls back to ask when no rule matches. — [packages/opencode/src/agent/agent.ts:119-121](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/agent/agent.ts#L119-L121); [packages/opencode/src/permission/index.ts:28-37](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/permission/index.ts#L28-L37) (verified)
  - *To reach the next level:* Default consequential tools (bash, edit, webfetch, MCP) to ask, with disabling requiring an explicit operator flag.
- **B L2:** File edits are recoverable via git snapshots stored outside the workspace (on by default), but shell-driven external actions (git push, network calls, deletes outside git) are irreversible. — [packages/opencode/src/snapshot/index.ts:167-170](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/snapshot/index.ts#L167-L170); [packages/opencode/src/snapshot/index.ts:71](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/snapshot/index.ts#L71) (verified)
  - *To reach the next level:* Add previews or dry-runs for external actions, not just filesystem rollback.
- **Cap:** C2-POWERBYPASS — The shell tool, the most powerful action path, executes without approval in the default configuration because the default rule allows bash.

### C3 Tool & action scoping — 0.28 (high)

The default tool set includes an unrestricted shell, file write/edit/patch, and a web fetcher that accepts any http(s) URL. File tools and the shell's file commands check whether a path lies inside the project, but the check is not a strict boundary and only triggers a prompt rather than a refusal. The fetcher has no host allowlist and no block on internal or metadata addresses. Tools can be disabled individually by setting deny rules, but everything is on by default.

- **S L1:** The shell takes a raw command string and webfetch any http(s) URL; path containment is not a strict boundary. — [packages/opencode/src/tool/webfetch.ts:35-48](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/webfetch.ts#L35-L48); searched `rg -n -i '169\.254|isPrivate|127\.0\.0\.1'` in `packages/opencode/src/tool` → 0 hits (No internal-address or metadata-IP blocking in any tool.) (verified)
  - *To reach the next level:* Validate arguments against allowlists in code: resolved-path containment, URL host allowlists with internal-address blocking.
- **C L2:** Most built-in file tools and the shell's file commands run the external-directory check; webfetch, MCP and plugin tools have no argument validation. — [packages/opencode/src/tool/external-directory.ts:15-44](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/external-directory.ts#L15-L44); [packages/opencode/src/tool/shell.ts:397-404](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L397-L404) (verified)
  - *To reach the next level:* Wrap extension tools in a shared validation layer.
- **D L1:** Shell, write, edit, webfetch and task are all enabled by default; individual tools can be hidden with a deny rule. — [packages/opencode/src/tool/registry.ts:231-246](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/registry.ts#L231-L246); [packages/opencode/src/permission/index.ts:204-213](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/permission/index.ts#L204-L213) (verified)
  - *To reach the next level:* Ship a read-only default tool set and require explicit enabling of write/exec.
- **B L0:** A misused shell tool reaches the whole machine with the user's privileges. — [SECURITY.md:15-17](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/SECURITY.md#L15-L17); [packages/opencode/src/tool/shell.ts:416-426](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L416-L426) (verified)
  - *To reach the next level:* Scope the general tools to the workspace and bound quantities.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

There is no isolation of any kind: shell commands, MCP stdio servers, custom tools and plugins run as ordinary processes (or in-process code) under the user's account with the full environment. The project's own security policy says plainly that it does not sandbox the agent and that the permission system is not a security boundary. Combined with the default allow-all rules, any command the model chooses runs on the host immediately.

- **S L0:** Commands run as same-user host subprocesses; the maintainers state there is no sandbox. — [SECURITY.md:15-17](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/SECURITY.md#L15-L17); [packages/opencode/src/tool/shell.ts:303-309](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L303-L309); searched `rg -n -i 'sandbox-exec|seccomp|landlock|bwrap|firejail|gvisor|firecracker'` in `packages/opencode/src packages/core/src` → 0 hits (No sandbox primitive anywhere in the CLI or core.) (verified)
  - *To reach the next level:* Provide an OS-level sandbox (Seatbelt/Landlock+seccomp or a hardened container) for command execution.
- **C L0:** No execution path is sandboxed (shell, MCP stdio, custom tools, plugins). — [SECURITY.md:15-17](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/SECURITY.md#L15-L17); [packages/opencode/src/mcp/index.ts:347-356](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/mcp/index.ts#L347-L356) (verified)
  - *To reach the next level:* Route at least the main shell tool through an isolation boundary.
- **D L0:** No sandbox exists to enable. — [SECURITY.md:15-17](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/SECURITY.md#L15-L17) (verified)
  - *To reach the next level:* Ship a sandbox on by default.
- **B L0:** Host-equivalent: the full home directory, credentials in the environment, and unrestricted network are reachable. — [packages/opencode/src/tool/shell.ts:416-426](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L416-L426); [SECURITY.md:15-17](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/SECURITY.md#L15-L17) (verified)
  - *To reach the next level:* Confine execution to the workspace with no secrets in the environment and egress controls.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Web pages, files in the repository, MCP tool results and MCP server instructions all enter the model's context with no marking or separation, and MCP servers' instructions are appended to the system prompt. Because the default rules allow shell and web fetch without approval, a successful prompt injection can read secrets and send them out (curl, webfetch to any URL) and also take irreversible actions such as pushing code or deleting files, with no human in the loop.

- **S L0:** Nothing structurally limits a hijacked session; there is no taint tracking or detection. — searched `rg -n -i 'untrusted|prompt.injection'` in `packages/opencode/src/session packages/opencode/src/tool` → 0 hits (No provenance or untrusted-content handling in the session loop or tools.); [packages/opencode/src/agent/agent.ts:119-121](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/agent/agent.ts#L119-L121) (verified)
  - *To reach the next level:* Disable or gate egress and state-changing tools once untrusted content has been read.
- **C L0:** Tool results and MCP server instructions enter context with the same standing as the user's input; MCP instructions are placed in the system prompt. — [packages/opencode/src/session/prompt.ts:1257-1268](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/session/prompt.ts#L1257-L1268) (verified)
  - *To reach the next level:* Distinguish untrusted sources (tool results, MCP output and descriptions) from principal input.
- **D L0:** No control exists to be on by default. — [packages/opencode/src/agent/agent.ts:119-121](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/agent/agent.ts#L119-L121) (verified)
  - *To reach the next level:* Ship a provenance-based restriction on by default.
- **B L0:** A hijacked session can exfiltrate (webfetch to any URL, shell curl) and take irreversible actions (shell), unattended, under the default allow-all ruleset. — [packages/opencode/src/agent/agent.ts:119-121](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/agent/agent.ts#L119-L121); [packages/opencode/src/tool/webfetch.ts:35-48](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/webfetch.ts#L35-L48); [packages/opencode/src/tool/shell.ts:416-426](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L416-L426) (verified)
  - *To reach the next level:* Require approval for egress and irreversible actions after untrusted input is read.
- **Cap:** C5-WORSTCASE — Default config lets a hijacked session both leak data and take irreversible actions without a human.

### C6 Memory, context & configuration integrity — 0.05 (high)

Opening OpenCode inside a repository silently loads that repository's configuration: opencode.json, the .opencode directory's agents, commands and config, plugins in .opencode/plugin(s), and custom tools in .opencode/tool(s), which are imported and executed in-process at startup. Project config can also declare MCP servers (launched automatically), npm plugins, permission rules, provider endpoints and automatic session sharing. AGENTS.md/CLAUDE.md instruction files are also loaded silently. There is no workspace-trust prompt; the only off switch is an environment variable. Because edits are allowed by default, a hijacked session can also write these files and plant a backdoor that fires in every later session and for anyone who clones the repo.

- **S L0:** Repo-controlled files add tools, plugins, MCP servers and permission rules with no prompt. — [packages/opencode/src/config/config.ts:420-424](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/config/config.ts#L420-L424); [packages/opencode/src/config/config.ts:438-479](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/config/config.ts#L438-L479); [packages/opencode/src/tool/registry.ts:183-192](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/registry.ts#L183-L192) (verified)
  - *To reach the next level:* Require an explicit workspace-trust decision before project config can add tools, plugins, MCP servers or change permissions.
- **C L0:** No auto-loaded path is controlled: project config, .opencode directories and instruction files all load unconditionally. — [packages/opencode/src/session/instruction.ts:64-68](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/session/instruction.ts#L64-L68); [packages/opencode/src/config/config.ts:438-479](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/config/config.ts#L438-L479); searched `rg -n -i 'trust'` in `packages/opencode/src/config` → 0 hits (No workspace-trust concept in the config loader.) (verified)
  - *To reach the next level:* Put every auto-loaded file and setting behind the same trust decision.
- **D L1:** Sessions are stored per project in a user-scope database, but the model can write project config and instruction files (edit allowed by default), altering future behaviour. — [packages/opencode/src/agent/agent.ts:119-121](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/agent/agent.ts#L119-L121); [packages/opencode/src/config/config.ts:420](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/config/config.ts#L420) (verified)
  - *To reach the next level:* Prevent the model from writing security-relevant config without approval.
- **B L0:** Poisoned project config persists across sessions and across every user who clones the repo, and can trigger code execution and tool use. — [packages/opencode/src/tool/registry.ts:183-192](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/registry.ts#L183-L192); [packages/opencode/src/plugin/loader.ts:139](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/plugin/loader.ts#L139) (verified)
  - *To reach the next level:* Make persisted config changes require human review and be easy to roll back.
- **Cap:** C6-REPOCONFIG — Project opencode.json and .opencode/ files load without any trust decision and can add plugins, custom tools, MCP servers and permission rules (config.ts:420-479, registry.ts:183-192).

### C7 Third-party extensions — 0.00 (high)

Extensions load with no verification and no consent. Plugin files in .opencode/plugin(s), custom tools in .opencode/tool(s), and npm plugins named in any config (including a repository's own opencode.json) are imported straight into the OpenCode process; npm plugins named without a version resolve to latest each time. MCP servers declared in config start automatically as child processes with the full environment. The one mitigation found is that npm install runs with lifecycle scripts disabled.

- **S L0:** Unverified code is executed automatically: npm plugins default to 'latest' and local plugin/tool files are dynamically imported. — [packages/opencode/src/plugin/shared.ts:32](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/plugin/shared.ts#L32); [packages/opencode/src/plugin/loader.ts:139](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/plugin/loader.ts#L139); [packages/core/src/npm.ts:100](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/core/src/npm.ts#L100) (Install scripts are skipped, but the package's code is still imported in-process.); searched `rg -n -i 'integrity|sha512'` in `packages/opencode/src/plugin packages/opencode/src/mcp` → 0 hits (No integrity or hash checks in plugin or MCP loading.) (verified)
  - *To reach the next level:* Pin extension versions and verify integrity before loading.
- **C L0:** No extension type (npm plugin, file plugin, custom tool, MCP server) is verified. — [packages/opencode/src/tool/registry.ts:183-192](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/registry.ts#L183-L192); [packages/opencode/src/mcp/index.ts:505-519](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/mcp/index.ts#L505-L519) (verified)
  - *To reach the next level:* Verify at least one extension type.
- **D L0:** Workspace files can add plugins, tools and MCP servers silently; configured MCP servers auto-start. — [packages/opencode/src/config/config.ts:420-424](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/config/config.ts#L420-L424); [packages/opencode/src/mcp/index.ts:505-519](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/mcp/index.ts#L505-L519); [packages/opencode/src/config/plugin.ts:21](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/config/plugin.ts#L21) (verified)
  - *To reach the next level:* Enable nothing third-party by default and show the exact package/command before first run.
- **B L0:** Plugins and custom tools run in-process; MCP stdio servers run as the same user with the full environment. — [packages/opencode/src/plugin/loader.ts:139](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/plugin/loader.ts#L139); [packages/opencode/src/mcp/index.ts:347-356](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/mcp/index.ts#L347-L356) (verified)
  - *To reach the next level:* Run extensions out of process with a scrubbed environment.
- **Cap:** C7-RCELOAD — By default, code from the workspace (.opencode/tool, .opencode/plugin) and npm plugins named in project config are imported and executed without consent.

### C8 Secrets & sensitive-data protection — 0.20 (high)

Provider credentials are stored in a plaintext auth.json with owner-only permissions, and the read tool asks before opening .env files. Beyond that there is no redaction: shell commands and MCP servers receive the whole environment, tool output goes to the model unfiltered, and the shell can read .env files without prompting. Telemetry is off unless an OpenTelemetry endpoint is configured, but a repository's own config can switch on automatic session sharing, which uploads transcripts to the share service.

- **S L1:** Secrets come from env vars and a 0600 plaintext auth.json; the only masking-like control is the read tool's ask for .env files. — [packages/opencode/src/auth/index.ts:79](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/auth/index.ts#L79); [packages/opencode/src/agent/agent.ts:130-135](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/agent/agent.ts#L130-L135); searched `rg -n -i 'redact|secretstr'` in `packages/opencode/src/session packages/opencode/src/tool` → 0 hits (No redaction in the session loop or tools.) (verified)
  - *To reach the next level:* Add redaction before logs and model-bound messages and store credentials in the OS keychain.
- **C L1:** Only the read tool's .env path is protected; subprocess environments, model-bound tool output, logs and transcripts are not. — [packages/opencode/src/tool/shell.ts:416-426](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L416-L426); [packages/opencode/src/agent/agent.ts:130-135](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/agent/agent.ts#L130-L135) (verified)
  - *To reach the next level:* Protect logs, transcripts and subprocess environments too.
- **D L1:** Telemetry export is off without an OTLP endpoint, but repo-controlled config can set share to 'auto' and upload every new session. — [packages/core/src/observability/otlp.ts:51](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/core/src/observability/otlp.ts#L51); [packages/opencode/src/share/session.ts:43](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/share/session.ts#L43) (verified)
  - *To reach the next level:* Prevent workspace config from enabling sharing and keep redaction always on.
- **B L0:** Long-lived provider keys and the user's ambient credentials are reachable by the model through every shell subprocess. — [packages/opencode/src/tool/shell.ts:416-426](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L416-L426) (verified)
  - *To reach the next level:* Keep long-lived keys out of subprocess environments.
- **Cap:** none

### C9 Audit & traceability — 0.50 (high)

Every session is stored in a SQLite database in the user's data directory, outside the project, with each tool call's name, input, status and timestamps recorded as it runs, including MCP tools and sub-agent sessions. Approval decisions are only published as transient events, not stored, and there is no actor attribution or tamper protection; with shell allowed by default the agent can edit or delete the database. Logs are local files.

- **S L2:** Structured per-tool-call records (tool, input, status, start time) are written to the session database. — [packages/opencode/src/session/processor.ts:336-347](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/session/processor.ts#L336-L347); [packages/core/src/session/sql.ts:82-92](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/core/src/session/sql.ts#L82-L92) (verified)
  - *To reach the next level:* Add actor attribution (approver, requesting principal) and correlation across sub-agents.
- **C L2:** All tool calls including MCP are recorded, but approvals and denials are not persisted. — [packages/opencode/src/permission/index.ts:115-119](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/permission/index.ts#L115-L119) (verified)
  - *To reach the next level:* Persist approval and denial decisions alongside tool calls.
- **D L2:** On by default and stored outside the workspace, but the agent's shell (allowed by default) can alter it. — [packages/core/src/database/database.ts:53](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/core/src/database/database.ts#L53); [packages/opencode/src/agent/agent.ts:119-121](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/agent/agent.ts#L119-L121) (verified)
  - *To reach the next level:* Write the record through a component the model cannot control.
- **B L2:** Tool-call state is written to the database at each transition, so records are flushed per action. — [packages/opencode/src/session/processor.ts:150](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/session/processor.ts#L150) (verified)
  - *To reach the next level:* Make the record durable per action with replay and fail closed for high-risk actions.
- **Cap:** none

### C10 Limits & kill switch — 0.35 (high)

The agent loop has no step limit by default (agent steps default to unlimited) and there is no cost or session time limit. Shell commands time out after two minutes by default, but the model can pass any larger timeout. Pressing stop aborts the loop and kills the running command's process group. A repeated-identical-call detector asks the user after three repeats, but that is a progress heuristic, not a budget.

- **S L2:** An optional per-agent step cap and a per-command shell timeout are enforced in code; abort kills the shell's process group. — [packages/opencode/src/session/prompt.ts:1178](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/session/prompt.ts#L1178); [packages/opencode/src/tool/shell.ts:540-555](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L540-L555); [packages/core/src/cross-spawn-spawner.ts:308](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/core/src/cross-spawn-spawner.ts#L308) (verified)
  - *To reach the next level:* Add wall-clock and token/cost caps plus rate limits on side-effecting tools.
- **C L2:** Limits cover the top-level loop (when steps is set) and shell timeouts; sub-agents and background tasks get their own fresh step counts. — [packages/opencode/src/tool/task.ts:58-60](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/task.ts#L58-L60); [packages/opencode/src/session/prompt.ts:1178](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/session/prompt.ts#L1178) (verified)
  - *To reach the next level:* Count sub-agents and background tasks against the parent's budget.
- **D L1:** Steps are unlimited by default and the model can raise the shell timeout without an upper bound. — [packages/opencode/src/session/prompt.ts:1178](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/session/prompt.ts#L1178); [packages/opencode/src/tool/shell.ts:615-618](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/tool/shell.ts#L615-L618) (verified)
  - *To reach the next level:* Ship sensible default step/time caps the model cannot raise.
- **B L0:** With no step or spend ceiling, a runaway session can loop and spend indefinitely. — [packages/opencode/src/session/prompt.ts:1178](https://github.com/anomalyco/opencode/blob/907b3bc518fa48e90e8ec24dd327d13eee71c36c/packages/opencode/src/session/prompt.ts#L1178); searched `rg -n -i 'maxCost|cost_limit|costLimit|max_cost'` in `packages/opencode/src packages/core/src` → 2 hits (Both hits are model-catalog ranking in core/src/catalog.ts, not a spend limit.) (verified)
  - *To reach the next level:* Add tight per-run time and cost ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: webfetch of any URL (tool/webfetch.ts:35-48), repo files, MCP results and MCP instructions (session/prompt.ts:1257-1268) · [B] sensitive data/systems: full process environment and home directory via the shell (tool/shell.ts:416-426) · [C] state change / egress: shell, edit/write and webfetch allowed without approval by default (agent/agent.ts:119-121) · Same default session? Yes

## Highest-impact improvements
1. Change the default build-agent rules so bash, edit/write, webfetch and MCP tools ask, matching what the docs describe. — C2 D L0→L3, +0.150 before caps (Playbook 5, step 1)
2. Add a workspace-trust prompt before loading project opencode.json, .opencode plugins/tools/agents and MCP servers from a repository. — C6 S L0→L3, +0.225 before caps (Playbook 2)
3. Require explicit consent showing the exact package/file/command before importing plugins, custom tools or starting MCP servers, and pin npm plugin versions. — C7 D L0→L3, +0.150 before caps (Playbook 3)
4. Pass a scrubbed environment to shell commands and MCP servers instead of the full process.env. — C1 C L0→L1, +0.075 before caps (Playbook 4)
5. Ship a default step cap and an upper bound on model-chosen shell timeouts. — C10 D L1→L2, +0.050 before caps (Playbook 3, step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scope was the packages/opencode CLI/TUI and the packages/core code it calls; the desktop app, web console, enterprise, Slack and GitHub-action packages were not examined.
- Server mode, the --auto/--dangerously-skip-permissions flags, and the experimental code-mode tool were not scored (non-default).
- Documentation/code mismatch: SECURITY.md:17 says the permission system prompts before commands, but the shipped default ruleset allows bash; a further mode-specific mismatch also exists.
- Whether background sub-agent tasks are cancelled when the parent session is aborted was not traced end to end.
- No reviewer-injection text was found in the repository's markdown files.
