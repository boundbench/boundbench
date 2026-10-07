# Defense-in-Depth Score: Crush

**Repo:** https://github.com/charmbracelet/crush · **Commit:** `bdcf796cb1ff241b0eb18139071d46a84dc1ee91` · **Reviewed:** 2026-10-03
**What it is:** Charm's glamorous terminal agentic coding tool
**Category:** Coding
**Scored configuration:** Interactive TUI ('crush' with no flags), fresh install, no MCP servers configured, opened in a project directory.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 1.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L0 | L0 | 0.07 | — | **0.07** | High |
| C2 | Approval gates | L3 | L1 | L1 | L1 | 0.40 | G2 | **0.25** | High |
| C3 | Tool & action scoping | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L2 | L0 | L1 | L0 | 0.20 | C5-WORSTCASE | **0.20** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | L1 | L0 | L0 | L1 | 0.12 | C7-RCELOAD | **0.12** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L2 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |


Crush asks before most shell commands and file writes, and shows the real command or diff. But the prompt does not cover every path, and a cloned repository can turn approval off or run its own shell script the moment you launch Crush there. Nothing is sandboxed, every command gets your full environment and credentials, and there are no step, time or cost limits. Treat it as running with your full user authority, and only launch it in repositories whose crushrc/crush.json you have reviewed.

## Critical gaps
- A repository's crush.json, .crushrc or .crush/crush.json can set permissions.allowed_tools to auto-approve bash, silently disabling the approval gate. (ASI09, ASI06, T10; C2) — [internal/permission/permission.go:186-190](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/permission/permission.go#L186-L190); [internal/app/app.go:104-107](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/app/app.go#L104-L107); [internal/config/load.go:960-976](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/load.go#L960-L976); [internal/config/load.go:66-67](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/load.go#L66-L67)
- A .crushrc or crushrc in the opened project is executed as a shell script at startup with the user's full environment and no trust prompt. (ASI06, ASI05, T1, T11; C6) — [internal/config/load.go:960-976](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/load.go#L960-L976); [internal/config/load.go:1228-1231](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/load.go#L1228-L1231); [internal/shellconfig/load.go:46-52](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/shellconfig/load.go#L46-L52)
- MCP servers declared in any loaded config, including the repository's, are launched automatically with the full environment and no consent. (ASI04, T17; C7) — [internal/agent/tools/mcp/init.go:342-354](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/mcp/init.go#L342-L354); [internal/agent/tools/mcp/init.go:1186-1187](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/mcp/init.go#L1186-L1187)
- A hijacked default session can read secrets (safe-listed printenv), and because the approval gate does not cover every path it can exfiltrate them and run destructive commands without any human approval. (ASI01, LLM01, T6; C5) — [internal/agent/tools/safe.go:26](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/safe.go#L26)
- No isolation: all commands run as the developer's user with home directory, credentials and network. (ASI05, T11; C4) — [internal/shell/shell.go:92-95](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/shell/shell.go#L92-L95); [internal/shellconfig/load.go:46-52](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/shellconfig/load.go#L46-L52)
- Ambient user identity with full environment passed to every subprocess; a bypassed gate reaches the whole user account. (ASI03, T3; C1) — [internal/shell/shell.go:92-95](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/shell/shell.go#L92-L95); [internal/agent/tools/mcp/init.go:1186-1187](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/mcp/init.go#L1186-L1187)

## Criterion details

### C1 Identity & least privilege — 0.07 (high)

Crush runs as the developer's own OS user and narrows none of that authority. Every shell command and every MCP server it launches inherits the full process environment, so model-provider API keys, cloud credentials, GitHub tokens and the SSH agent are all reachable. There is no per-tool identity or credential scoping; the only check between the model and those credentials is the approval prompt, which does not cover every path. A hijacked session therefore acts with everything the developer can do.

- **S L0:** The agent uses the operator's ambient OS identity and environment with no narrowing; the shell receives os.Environ() unchanged apart from two herdr pane variables. — [internal/shell/shell.go:92-95](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/shell/shell.go#L92-L95) (verified)
  - *To reach the next level:* No scrubbing of credential variables or scoped tokens for tool subprocesses.
- **C L1:** Every bash and MCP stdio subprocess gets the full environment; only the human approval prompt (scored in C2) stands between tools and ambient credentials. — [internal/shell/shell.go:92-95](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/shell/shell.go#L92-L95); [internal/agent/tools/mcp/init.go:1186-1187](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/mcp/init.go#L1186-L1187) (verified)
  - *To reach the next level:* MCP servers and shell commands should receive a scrubbed environment instead of os.Environ().
- **D L0:** A fresh install runs with the developer's complete privileges; there is no narrower default profile. — [internal/shell/shell.go:92-95](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/shell/shell.go#L92-L95) (verified)
  - *To reach the next level:* No default least-privilege mode (e.g. env allowlist) for subprocesses.
- **B L0:** With the approval layer bypassed (e.g. repo-set allowed_tools), commands reach the user's whole account: SSH keys, cloud CLIs, gh auth, and the home directory. — [internal/shell/shell.go:92-95](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/shell/shell.go#L92-L95) (verified)
  - *To reach the next level:* Credentials reachable from tools should be limited to one project/system.
- **Cap:** none

### C2 Approval gates — 0.25 (high)

By default Crush asks before running shell commands, writing or editing files, fetching URLs and calling MCP tools, and the prompt shows the real command or diff. But the gate has several holes. The built-in 'safe command' list is not a strict boundary. A project's own crush.json or .crushrc can add bash to allowed_tools, which silently auto-approves it. 'Allow for session' on bash approves every later command in that directory, and an approved agentic_fetch sub-agent runs with its whole session auto-approved.

- **S L3:** Per-call approval shows the exact command, parameters and file diffs, with a built-in safe-command tier deciding what skips the prompt. — [internal/agent/tools/bash.go:228-239](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/bash.go#L228-L239); [internal/agent/tools/write.go:108-120](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/write.go#L108-L120) (verified)
  - *To reach the next level:* No argument-level policy on parsed commands; 'Allow for Session' is keyed on tool+action+directory, not on the command.
- **C L1:** The safe-command list is not a strict boundary; sourcegraph egress is never gated and agentic_fetch sub-sessions are auto-approved. — searched `rg -n permission` in `internal/agent/tools/sourcegraph.go` → 0 hits (The sourcegraph tool never calls the permission service; it posts the model-chosen query to sourcegraph.com.); [internal/agent/agentic_fetch_tool.go:199-201](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/agentic_fetch_tool.go#L199-L201); [internal/permission/permission.go:248-253](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/permission/permission.go#L248-L253) (verified)
  - *To reach the next level:* Auto-approved commands must be a verified read-only allowlist, and every egress tool and sub-agent must cross the gate.
- **D L1:** On by default, but permissions.allowed_tools from a project crush.json, .crushrc, or .crush/crush.json in the workspace silently auto-approves tools such as bash; --yolo and non-interactive 'crush run' approve everything. — [internal/permission/permission.go:186-190](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/permission/permission.go#L186-L190); [internal/app/app.go:104-107](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/app/app.go#L104-L107); [internal/config/load.go:960-976](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/load.go#L960-L976); [internal/config/load.go:66-67](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/load.go#L66-L67); [internal/app/app.go:384-386](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/app/app.go#L384-L386); [internal/cmd/root.go:60](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/cmd/root.go#L60) (verified)
  - *To reach the next level:* Allow-rules must come only from user scope, never from files in the repository.
- **B L1:** Unapproved or wrongly approved shell commands can delete files, push code or send data; file history exists for edits but nothing checkpoints shell effects. — [internal/agent/tools/bash.go:228-239](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/bash.go#L228-L239); [internal/agent/coordinator.go:875-893](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/coordinator.go#L875-L893) (verified)
  - *To reach the next level:* No checkpoint/rollback of shell or filesystem effects beyond the edit history.
- **Cap:** G2 — A file in the workspace (crush.json, .crushrc, .crush/crush.json) can set permissions.allowed_tools to auto-approve bash at runtime.

### C3 Tool & action scoping — 0.20 (high)

Crush's main tool is an arbitrary shell, guarded only by a denylist of command names (curl, wget, ssh, sudo, package installs) that is not a complete boundary. File tools accept any path on the machine and rely on the approval prompt; view and ls only ask when a path is outside the working directory, and that check is not a strict boundary. fetch and download accept any http(s) URL including localhost and cloud metadata addresses. All tools, including shell, write and network, are enabled by default, though each can be disabled in config.

- **S L1:** Bash is filtered by a command-name denylist; other tools do only light checks (http/https scheme, outside-workdir prompt), and none of these checks is a complete boundary. — [internal/agent/tools/bash.go:76-83](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/bash.go#L76-L83); [internal/agent/tools/fetch.go:71](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/fetch.go#L71) (verified)
  - *To reach the next level:* No allowlist validation: no path containment, no internal-address block or redirect recheck on URLs.
- **C L1:** Only a few tools validate anything (view/ls outside-dir prompt, fetch/download scheme); MCP tools pass arguments through unchecked. — [internal/agent/tools/fetch.go:71](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/fetch.go#L71) (verified)
  - *To reach the next level:* Most built-in tools should validate arguments, not only view/ls/fetch.
- **D L1:** Every built-in tool, including bash, write and network fetch, is enabled by default; disabled_tools can remove them individually. — [internal/config/config.go:1059-1063](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/config.go#L1059-L1063); [internal/agent/coordinator.go:875-893](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/coordinator.go#L875-L893) (verified)
  - *To reach the next level:* Write/exec tool groups enabled by default; no read-only default set.
- **B L0:** A misused bash tool runs any command against the whole machine as the user. — [internal/agent/coordinator.go:875-893](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/coordinator.go#L875-L893) (verified)
  - *To reach the next level:* Tools should be scoped to the project workspace.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

There is no isolation at all. Shell commands run through an embedded Go shell interpreter that executes real binaries as the developer's user, in the developer's environment, with full network access. Repository-supplied crushrc files, hooks, LSP servers and MCP servers also run directly on the host. A denylist of command names is the only filter, and it is not a complete boundary.

- **S L0:** Commands execute as same-user host processes via mvdan.cc/sh with a process-group exec handler; no sandbox primitive exists. — searched `rg -n -S -g !*_test.go 'landlock|seatbelt|sandbox-exec|bwrap|bubblewrap|seccomp|nsjail|firejail|gvisor|runsc|firecracker'` in `internal` → 0 hits (No isolation primitive anywhere in the Go source.); [internal/shell/shell.go:92-95](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/shell/shell.go#L92-L95) (verified)
  - *To reach the next level:* No OS sandbox or container around command execution.
- **C L0:** No execution path is sandboxed: bash, background jobs, hooks, crushrc config scripts and MCP stdio servers all run on the host. — searched `rg -n -S -g !*_test.go 'landlock|seatbelt|sandbox-exec|bwrap|bubblewrap|seccomp|nsjail|firejail|gvisor|runsc|firecracker'` in `internal` → 0 hits (No isolation primitive anywhere in the Go source.); [internal/shellconfig/load.go:46-52](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/shellconfig/load.go#L46-L52); [internal/agent/tools/mcp/init.go:1186-1187](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/mcp/init.go#L1186-L1187) (verified)
  - *To reach the next level:* Some execution path would need to run inside an isolation boundary.
- **D L0:** No sandbox exists to be on by default. — searched `rg -n -S -g !*_test.go 'landlock|seatbelt|sandbox-exec|bwrap|bubblewrap|seccomp|nsjail|firejail|gvisor|runsc|firecracker'` in `internal` → 0 hits (No isolation primitive anywhere in the Go source.) (verified)
  - *To reach the next level:* No default-on sandbox.
- **B L0:** Executed code has the user's home directory, ~/.ssh and cloud credentials, all environment secrets and unrestricted network. — [internal/shell/shell.go:92-95](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/shell/shell.go#L92-L95); searched `rg -n -S -g !*_test.go 'landlock|seatbelt|sandbox-exec|bwrap|bubblewrap|seccomp|nsjail|firejail|gvisor|runsc|firecracker'` in `internal` → 0 hits (No isolation primitive anywhere in the Go source.) (verified)
  - *To reach the next level:* Execution should be limited to the workspace without secrets or open egress.
- **Cap:** none

### C5 Untrusted input blast radius — 0.20 (high)

Crush reads files, web pages and MCP results straight into the model context with no marking of what is untrusted and nothing that changes behaviour after reading it. The approval prompt is the only brake: printenv is on the safe list, so a hijacked model can read API keys from the environment unprompted, and the gate does not cover every path, so it can act on them without approval. The sourcegraph tool also sends model-chosen text to a third party without asking.

- **S L2:** Most state-changing and egress tools ask for approval, but not consistently: sourcegraph egress does not, and the shell gate does not cover every path. — [internal/agent/tools/bash.go:228-239](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/bash.go#L228-L239); searched `rg -n permission` in `internal/agent/tools/sourcegraph.go` → 0 hits (The sourcegraph tool never calls the permission service; it posts the model-chosen query to sourcegraph.com.) (verified)
  - *To reach the next level:* Once untrusted content is read, every egress and state change should be forced through approval (Rule of Two in code).
- **C L0:** Untrusted sources are not distinguished; tool results enter context as ordinary tool messages. — searched `rg -n -S -g !*_test.go -g !**/testdata/** 'untrusted|taint|provenance|quarantin'` in `internal` → 7 hits (Hits: an SVG blob, 'uncertainty' in a template and a markdown comment, a JWT-claims comment, a coordinator comment, and two comments in mcp/channel.go about escaping opt-in MCP channel payloads. None marks tool results or files as untrusted for the main loop.); [internal/agent/agent.go:1021-1038](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/agent.go#L1021-L1038) (verified)
  - *To reach the next level:* Tool/web/file results should carry an untrusted marking that drives policy.
- **D L1:** The approval prompt is on by default but a repository config file can auto-approve tools. — [internal/permission/permission.go:186-190](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/permission/permission.go#L186-L190); [internal/config/load.go:960-976](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/load.go#L960-L976) (verified)
  - *To reach the next level:* Workspace content must not be able to switch approval off.
- **B L0:** A hijacked default session can read secrets (printenv, view of workspace files) and exfiltrate them plus run irreversible commands through paths the approval gate does not cover, with no human involved. — [internal/agent/tools/safe.go:26](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/safe.go#L26) (verified)
  - *To reach the next level:* Either exfiltration or irreversible action should require a human.
- **Cap:** C5-WORSTCASE — Leak plus irreversible action is possible unattended in the default configuration.

### C6 Memory, context & configuration integrity — 0.10 (high)

Opening Crush in a cloned repository trusts that repository completely. A .crushrc or crushrc in the project is executed as a shell script at startup, and $(...) in a project crush.json runs at load time, before any UI appears; the project's config can also add hooks, MCP servers and LSP servers, auto-approve tools, and redirect provider endpoints. The project's .crush/crush.json is merged with the highest priority. Instruction files such as AGENTS.md, CLAUDE.md and .cursorrules are loaded silently. The docs warn users about this, but the code asks for no trust decision.

- **S L0:** Repository-controlled files execute code and add hooks, MCP servers and auto-approve rules with no prompt. — [internal/config/load.go:960-976](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/load.go#L960-L976); [internal/config/load.go:1228-1231](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/load.go#L1228-L1231); [internal/shellconfig/load.go:46-52](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/shellconfig/load.go#L46-L52); [internal/config/load.go:66-67](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/load.go#L66-L67); [internal/permission/permission.go:186-190](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/permission/permission.go#L186-L190) (verified)
  - *To reach the next level:* Security-relevant project config needs an explicit workspace-trust decision.
- **C L0:** No config or context path is controlled: crushrc, crush.json, .crush/crush.json, project skills directories and instruction files all load silently. — [internal/config/load.go:960-976](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/load.go#L960-L976); [internal/config/config.go:29-46](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/config.go#L29-L46) (verified)
  - *To reach the next level:* Auto-loaded files and settings need a gate.
- **D L1:** Sessions are stored per project in the local .crush data directory (single-user local state), but the model can write that directory, including .crush/crush.json, via shell; rated L1 because the mechanism itself is L0. — [internal/config/config.go:25](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/config.go#L25); [internal/db/connect.go:93](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/db/connect.go#L93); [internal/config/load.go:66-67](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/load.go#L66-L67) (verified)
  - *To reach the next level:* The model should not be able to write the data directory that holds config and sessions.
- **B L1:** A poisoned crushrc, crush.json or AGENTS.md persists across all the user's sessions in that project and can run code or trigger tool use. — [internal/shellconfig/load.go:46-52](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/shellconfig/load.go#L46-L52); [internal/config/config.go:29-46](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/config.go#L29-L46) (verified)
  - *To reach the next level:* Poisoned context should only influence text or gated actions.
- **Cap:** C6-REPOCONFIG — Workspace .crushrc/crushrc is executed and project crush.json/.crush/crush.json can add hooks, MCP servers, auto-approve rules and provider endpoints with no trust prompt.

### C7 Third-party extensions — 0.12 (high)

Crush launches MCP servers listed in any loaded config, including a repository's own crush.json, automatically at startup with no consent prompt. Commands are run exactly as written (often 'npx' or 'uvx' of an unpinned package), with no hash or signature check, and each server gets the full user environment. MCP tool calls are approval-gated, but by then the server process already runs with the user's authority. When the Docker MCP gateway is enabled, the model can add new MCP servers without a prompt.

- **S L1:** MCP servers are user/config-chosen commands with no version pinning or integrity verification enforced by Crush. — searched `rg -n -S -g !*_test.go 'sha256|checksum|integrity|signature'` in `internal/agent/tools/mcp internal/config/mcp.go` → 0 hits (No hash or signature check on MCP server commands or packages.); [internal/agent/tools/mcp/init.go:1186-1187](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/mcp/init.go#L1186-L1187) (verified)
  - *To reach the next level:* Crush does not pin or verify extension versions.
- **C L0:** No extension type (MCP stdio/http, project skills, LSP servers) is verified. — searched `rg -n -S -g !*_test.go 'sha256|checksum|integrity|signature'` in `internal/agent/tools/mcp internal/config/mcp.go` → 0 hits (No hash or signature check on MCP server commands or packages.); [internal/agent/tools/mcp/init.go:342-354](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/mcp/init.go#L342-L354) (verified)
  - *To reach the next level:* At least one extension type should be verified.
- **D L0:** Any configured MCP server, including one added by a workspace config file, starts automatically without consent. — [internal/agent/tools/mcp/init.go:342-354](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/mcp/init.go#L342-L354); [internal/config/load.go:960-976](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/load.go#L960-L976); [internal/agent/tools/mcp-tools.go:15-21](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/mcp-tools.go#L15-L21) (verified)
  - *To reach the next level:* Adding an extension should require an explicit user action showing the command.
- **B L1:** Each MCP server is a separate process as the same user with the full os.Environ() plus its configured env. — [internal/agent/tools/mcp/init.go:1186-1187](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/mcp/init.go#L1186-L1187) (verified)
  - *To reach the next level:* Extension processes should receive a scrubbed environment.
- **Cap:** C7-RCELOAD — Opening a repository whose config lists an MCP server (or crushrc) runs that code, possibly fetched remotely via npx/uvx, with no consent.

### C8 Secrets & sensitive-data protection — 0.20 (high)

API keys come from environment variables or plaintext JSON config files written with 0600 permissions; there is no keychain use. Debug HTTP logging redacts auth-like headers but writes full request and response bodies. Every shell subprocess inherits the full environment, and printenv/env are on the unprompted safe list, so the model can read provider and cloud keys into its context. Anonymous usage telemetry to PostHog is on by default (no prompt content), and can be disabled via env var or config.

- **S L1:** Secrets come from env vars or plaintext config; masking exists only for auth headers in the debug HTTP logger. — [internal/log/http.go:104-111](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/log/http.go#L104-L111); [internal/config/store.go:327](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/store.go#L327); searched `rg -n -S -g !*_test.go 'keyring|keychain'` in `internal` → 0 hits (No OS keychain use; credentials live in JSON config files.) (verified)
  - *To reach the next level:* No type-level masking or redaction before tool output reaches the model.
- **C L1:** Only the debug HTTP header path is redacted; subprocess env, model-bound tool output and transcripts are not. — [internal/log/http.go:104-111](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/log/http.go#L104-L111); [internal/shell/shell.go:92-95](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/shell/shell.go#L92-L95) (verified)
  - *To reach the next level:* Logs and transcripts should both be redacted.
- **D L1:** Content-free PostHog telemetry is on by default (opt-out); debug logging is off by default but is not body-redacted. — [internal/event/event.go:16-18](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/event/event.go#L16-L18); [internal/cmd/root.go:905-915](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/cmd/root.go#L905-L915) (verified)
  - *To reach the next level:* Telemetry should be opt-in.
- **B L0:** Long-lived provider and cloud keys in the environment are reachable by every subprocess and readable by the model via safe-listed printenv. — [internal/agent/tools/safe.go:26](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/safe.go#L26); [internal/shell/shell.go:92-95](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/shell/shell.go#L92-L95) (verified)
  - *To reach the next level:* Keys reachable by tools should be scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.45 (high)

Every tool call and its result are saved as messages in a local SQLite database, written as each result arrives, so a session's actions can be reconstructed. Sub-agent runs are stored as their own sessions. Approvals and denials are not stored, and the database and logs live in the project's .crush directory, which the agent's own shell can edit or delete.

- **S L2:** Structured per-session transcript of tool calls and results in crush.db. — [internal/agent/agent.go:1021-1038](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/agent.go#L1021-L1038); [internal/db/connect.go:93](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/db/connect.go#L93) (verified)
  - *To reach the next level:* No actor attribution (approver, automatic vs human approval) on records.
- **C L2:** All tool calls including MCP tools are stored through the same message path; permission decisions are only published on an in-memory pubsub. — [internal/agent/agent.go:1021-1038](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/agent.go#L1021-L1038); [internal/agent/hooked_tool.go:82-84](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/hooked_tool.go#L82-L84) (verified)
  - *To reach the next level:* Approvals and denials are not persisted.
- **D L1:** On by default but stored in <project>/.crush, writable by the agent's own tools. — [internal/config/config.go:25](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/config/config.go#L25); [internal/db/connect.go:93](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/db/connect.go#L93) (verified)
  - *To reach the next level:* Records should be stored outside the workspace.
- **B L2:** Tool results are written to the database per result, and write errors propagate to the loop. — [internal/agent/agent.go:1021-1038](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/agent.go#L1021-L1038) (verified)
  - *To reach the next level:* No replay-grade durability guarantee or fail-closed recording.
- **Cap:** none

### C10 Limits & kill switch — 0.00 (high)

Crush has no cap on steps, time or cost per run. The only automatic stop is a loop detector for identical repeated tool calls and a context-window summarizer, neither of which bounds damage. Shell commands are not timed out: after 60 seconds they move to a background job that keeps running. Pressing cancel kills a foreground command's process group, but background jobs keep running until the app exits.

- **S L0:** No iteration, wall-clock, or token/cost limit; stop conditions are loop detection and auto-summarize only. — searched `rg -n -S -g !*_test.go 'MaxSteps|maxSteps|StepCountIs|max_steps|MaxIterations|max_iterations|MaxCost|max_cost|maxTurns|MaxTurns'` in `internal` → 0 hits (No step, turn, or cost cap exists in the agent loop.); [internal/agent/agent.go:1095-1119](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/agent.go#L1095-L1119); [internal/agent/loop_detection.go:12-13](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/loop_detection.go#L12-L13) (verified)
  - *To reach the next level:* Add at least an iteration cap plus a time or cost cap enforced in code.
- **C L0:** No limits exist to apply to the loop, sub-agents, or background jobs (beyond a 50-job count). — searched `rg -n -S -g !*_test.go 'MaxSteps|maxSteps|StepCountIs|max_steps|MaxIterations|max_iterations|MaxCost|max_cost|maxTurns|MaxTurns'` in `internal` → 0 hits (No step, turn, or cost cap exists in the agent loop.); [internal/shell/background.go:17](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/shell/background.go#L17) (verified)
  - *To reach the next level:* Limits should apply to the top-level loop.
- **D L0:** Unlimited by default. — searched `rg -n -S -g !*_test.go 'MaxSteps|maxSteps|StepCountIs|max_steps|MaxIterations|max_iterations|MaxCost|max_cost|maxTurns|MaxTurns'` in `internal` → 0 hits (No step, turn, or cost cap exists in the agent loop.) (verified)
  - *To reach the next level:* Sensible default limits.
- **B L0:** A runaway can loop and spend indefinitely; long commands auto-background on a detached context and survive cancellation. — [internal/agent/tools/bash.go:319-321](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/bash.go#L319-L321); [internal/agent/tools/bash.go:310](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/bash.go#L310); [internal/agent/tools/bash.go:338-342](https://github.com/charmbracelet/crush/blob/bdcf796cb1ff241b0eb18139071d46a84dc1ee91/internal/agent/tools/bash.go#L338-L342) (verified)
  - *To reach the next level:* Ceilings on run time/spend and cancellation of background work on stop.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Files, web fetch and MCP results enter context unmarked (internal/agent/agent.go:1021) · [B] sensitive data/systems: Full user environment incl. API keys readable via safe-listed printenv (internal/agent/tools/safe.go:26, internal/shell/shell.go:94) · [C] state change / egress: Bash, whose approval gate does not cover every path, sourcegraph egress (internal/agent/tools/sourcegraph.go:113) · Same default session? Yes

## Highest-impact improvements
1. Narrow the safe-command list to a minimal, verified read-only set (no printenv) and harden how it is enforced. — C2 C L1→L2, +0.075 before caps (Playbook 5)
2. Ignore permissions.allowed_tools, hooks, MCP servers and crushrc from project scope unless the user explicitly trusts the workspace. — C2 D L1→L3, +0.100 before caps (Playbook 5)
3. Add a workspace-trust prompt before executing a project .crushrc/crushrc or $(...) in project crush.json. — C6 S L0→L3, +0.225 before caps (Playbook 2)
4. Pass a scrubbed environment (no *_API_KEY, *_TOKEN, cloud credentials) to shell and MCP subprocesses. — C1 C L1→L2, +0.075 before caps (Playbook 4)
5. Add a default per-run step cap and wall-clock/cost ceiling, and kill background jobs on cancel. — C10 S L0→L2, +0.150 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was built, installed, or run.
- Score describes the interactive TUI default; non-interactive 'crush run' auto-approves every permission (internal/app/app.go:386) and --yolo skips all prompts, both worse than scored.
- The client/server split (internal/server, internal/backend), the TUI rendering of model output, OAuth flows, and LSP client internals were not examined in depth.
- Some shell-gate behaviour is inferred from the mvdan.cc/sh library's exec-handler design and was not observed at runtime.
- No reviewer-injection text was found in AGENTS.md, README.md or docs/.
