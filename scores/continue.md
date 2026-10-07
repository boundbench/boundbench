# Defense-in-Depth Score: Continue

**Repo:** https://github.com/continuedev/continue · **Commit:** `5522c6f44ca0ac3528b37244818fbfa39b5af470` · **Reviewed:** 2026-10-03
**What it is:** Open-source coding agent for IDEs and CLI (cn)
**Category:** Coding
**Scored configuration:** Continue CLI (cn) in interactive TUI mode, normal permission mode, no flags, fresh install with no ~/.continue/permissions.yaml; headless -p mode and the VS Code/JetBrains extensions are footnoted, not scored.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication yes

## Score: 2.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L2 | L1 | L2 | L1 | 0.38 | G2 | **0.25** | High |
| C3 | Tool & action scoping | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L2 | L0 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L1 | 0.30 | C6-REPOCONFIG | **0.25** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L1 | 0.23 | — | **0.23** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |


Continue's CLI asks before shell commands, file writes and MCP calls, but that approval gate does not cover every path. Read and Fetch also run unprompted with no workspace or host limits, giving a hijacked session an unattended exfiltration path. Everything runs on the host as the user, with the full environment, no sandbox, and no step or cost limits.

## Critical gaps
- A hijacked session can read files via the auto-approved Read tool and send them out via the auto-approved Fetch tool, with no human involved. (ASI01, LLM01, T6; C5) — [extensions/cli/src/permissions/defaultPolicies.ts:23](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/defaultPolicies.ts#L23); [extensions/cli/src/permissions/defaultPolicies.ts:21](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/defaultPolicies.ts#L21); [core/context/providers/URLContextProvider.ts:36-38](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/core/context/providers/URLContextProvider.ts#L36-L38)
- All commands and MCP servers run on the host as the user with the full process environment; there is no sandbox. (ASI05, T11, LLM05; C4) — [extensions/cli/src/tools/runTerminalCommand.ts:85-86](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L85-L86); [extensions/cli/src/tools/runTerminalCommand.ts:192](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L192); [extensions/cli/src/services/mcpTransports.ts:77-90](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/services/mcpTransports.ts#L77-L90)
- The agent holds the user's entire ambient authority (all env credentials, SSH agent, CLI logins) with no narrowing for subprocesses. (ASI03, T3; C1) — [extensions/cli/src/tools/runTerminalCommand.ts:192](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L192); [extensions/cli/src/services/mcpTransports.ts:77-90](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/services/mcpTransports.ts#L77-L90)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

The CLI runs as the logged-in user and does nothing to narrow that authority. Shell commands start a login shell that inherits the full process environment, and MCP servers are launched with the full environment merged into their own settings, so every cloud, GitHub, SSH-agent or API credential the user has is available to anything the agent runs. There is no per-tool identity or authorization layer; the only control is the approval prompt scored under approval gates, which does not cover every path.

- **S L0:** Bash runs the user's login shell with the inherited environment; no scoped identity or credential narrowing exists. — [extensions/cli/src/tools/runTerminalCommand.ts:85-86](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L85-L86); [extensions/cli/src/tools/runTerminalCommand.ts:192](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L192) (verified)
  - *To reach the next level:* No credential narrowing for subprocesses (env scrubbing, scoped tokens) or dedicated identity.
- **C L0:** No tool path narrows identity: Bash and MCP stdio servers receive the ambient environment. — [extensions/cli/src/tools/runTerminalCommand.ts:192](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L192); [extensions/cli/src/services/mcpTransports.ts:77-90](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/services/mcpTransports.ts#L77-L90) (verified)
  - *To reach the next level:* Not even the main tool path uses a narrowed identity or authorization check.
- **D L0:** Default install runs with the user's full ambient privileges; there is no least-privilege default to harden from. — [extensions/cli/src/tools/runTerminalCommand.ts:85-86](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L85-L86) (verified)
  - *To reach the next level:* No reduced-privilege default (e.g. scrubbed environment) for spawned processes.
- **B L0:** A hijacked session reaches everything the user's account can reach (cloud CLIs, gh auth, SSH agent, API keys in env). — [extensions/cli/src/tools/runTerminalCommand.ts:192](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L192); [extensions/cli/src/permissions/defaultPolicies.ts:24](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/defaultPolicies.ts#L24) (verified)
  - *To reach the next level:* Reachable authority is not limited to one system or to read-only access.
- **Cap:** none

### C2 Approval gates — 0.25 (high)

Interactive mode asks before running shell commands, writing or editing files, and calling MCP tools, and unknown tools default to asking. File edits show a real diff; the shell approval display does not always reflect exactly what will run. Choosing 'don't ask again' on one command writes a broad prefix rule such as Bash(git*) to the user's permissions file. The approval gate also does not cover every path. Read and Fetch also run unprompted, and there are no file checkpoints to undo damage.

- **S L2:** Per-call approval exists, with a full diff for Write/Edit; the Bash approval display does not always reflect exactly what will run. — [extensions/cli/src/tools/writeFile.ts:60-66](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/writeFile.ts#L60-L66) (verified)
  - *To reach the next level:* Bash approval display needs hardening.
- **C L1:** Bash, Write, Edit and MCP tools are gated, but the gate does not cover every path, and Fetch performs outbound requests unprompted. — [extensions/cli/src/permissions/defaultPolicies.ts:24](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/defaultPolicies.ts#L24); [extensions/cli/src/permissions/defaultPolicies.ts:21](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/defaultPolicies.ts#L21) (verified)
  - *To reach the next level:* A consequential path (network egress via Fetch) reaches execution without crossing the gate.
- **D L2:** Approval is on by default in the TUI; persisted allow rules live in user scope but 'don't ask again' silently widens one approval into a Bash(<cmd>*) prefix rule, and --auto or Shift+Tab mode cycling removes all prompts. — [extensions/cli/src/permissions/defaultPolicies.ts:34-37](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/defaultPolicies.ts#L34-L37); [extensions/cli/src/ui/components/ToolPermissionSelector.tsx:40-46](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/ui/components/ToolPermissionSelector.tsx#L40-L46); [extensions/cli/src/permissions/policyWriter.ts:25-29](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/policyWriter.ts#L25-L29); [extensions/cli/src/permissions/permissionsYamlLoader.ts:11-13](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/permissionsYamlLoader.ts#L11-L13); [extensions/cli/src/shared-options.ts:15](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/shared-options.ts#L15) (verified)
  - *To reach the next level:* Allow rules accumulate as broad prefix wildcards without being shown, rather than exact, visible, scoped rules.
- **B L1:** Shell commands can do irreversible things (push, delete, send); only a small denylist of critical commands (rm -rf /, mkfs, ...) is always blocked, and the CLI has no file checkpoints. — [packages/terminal-security/src/evaluateTerminalCommandSecurity.ts:374-377](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/packages/terminal-security/src/evaluateTerminalCommandSecurity.ts#L374-L377); [extensions/cli/src/tools/runTerminalCommand.ts:192](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L192) (verified)
  - *To reach the next level:* No checkpoint/rollback of filesystem or code state for the common case.
- **Cap:** G2 — The approval gate does not cover every runtime path.
- **Notes:** Headless mode (-p) auto-allows Bash and every MCP tool (defaultPolicies.ts:31-33); plan mode (--readonly) auto-allows Bash. The beta Subagent tool runs its child with all tools allowed (subagent/executor.ts:78-89).

### C3 Tool & action scoping — 0.20 (high)

The default tool set includes an arbitrary shell, arbitrary-URL fetch, and file read/write anywhere on disk. Argument checks are denylists: a list of always-blocked shell commands and a list of secret-looking file names that Read and Edit refuse. Read has no workspace containment, and Fetch accepts any URL including localhost and cloud metadata addresses. Tools can be excluded or a read-only plan mode chosen, but the default enables everything.

- **S L1:** Validation is denylist-based: critical shell commands are disabled and secret-looking filenames are refused, but Bash takes a raw command string and Fetch an arbitrary URL. — [packages/terminal-security/src/evaluateTerminalCommandSecurity.ts:374-377](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/packages/terminal-security/src/evaluateTerminalCommandSecurity.ts#L374-L377); [extensions/cli/src/tools/readFile.ts:56](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/readFile.ts#L56); [core/indexing/ignore.ts:246-254](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/core/indexing/ignore.ts#L246-L254); [core/context/providers/URLContextProvider.ts:36-38](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/core/context/providers/URLContextProvider.ts#L36-L38) (verified)
  - *To reach the next level:* No allowlist validation (resolved-path containment to the workspace, URL host allowlist blocking internal addresses).
- **C L1:** Only Bash (critical-command denylist), Read and Edit (secret-file denylist) validate; Fetch, Write, List, Search and all MCP tools do not. — [extensions/cli/src/tools/edit.ts:33-39](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/edit.ts#L33-L39); [extensions/cli/src/tools/fetch.ts:59](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/fetch.ts#L59); [extensions/cli/src/tools/index.tsx:137-140](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/index.tsx#L137-L140) (verified)
  - *To reach the next level:* Most built-in tools and all extension tools lack argument validation.
- **D L1:** Write, Bash, Fetch and MCP tools are all enabled by default; they can be individually excluded via --exclude or ~/.continue/permissions.yaml. — [extensions/cli/src/permissions/defaultPolicies.ts:21](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/defaultPolicies.ts#L21); [extensions/cli/src/permissions/defaultPolicies.ts:34-37](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/defaultPolicies.ts#L34-L37); [extensions/cli/src/permissions/permissionsYamlLoader.ts:11-13](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/permissionsYamlLoader.ts#L11-L13) (verified)
  - *To reach the next level:* The default tool group still includes write, exec and network tools; no read-only default.
- **B L0:** A misused tool can run any command or read any file the user can, and fetch any host. — [extensions/cli/src/tools/runTerminalCommand.ts:192](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L192); [extensions/cli/src/tools/readFile.ts:83-84](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/readFile.ts#L83-L84); [core/context/providers/URLContextProvider.ts:36-38](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/core/context/providers/URLContextProvider.ts#L36-L38) (verified)
  - *To reach the next level:* Tools are not scoped to the project/workspace.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

There is no execution isolation. Shell commands and MCP stdio servers all run directly on the host as the user, with the full environment, network and home directory available. No sandbox, container or OS profile exists anywhere in the CLI.

- **S L0:** Commands run as a same-user host subprocess via the login shell. — [extensions/cli/src/tools/runTerminalCommand.ts:85-86](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L85-L86); [extensions/cli/src/tools/runTerminalCommand.ts:192](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L192); searched `rg -n -i "sandbox|seccomp|landlock|seatbelt|bwrap|firejail|sandbox-exec"` in `extensions/cli/src` → 0 hits (No sandbox or OS isolation primitive anywhere in the CLI.) (verified)
  - *To reach the next level:* No OS-level isolation primitive (container, low-privilege user, Seatbelt/Landlock profile).
- **C L0:** No execution path is isolated: Bash and MCP stdio servers run on the host. — [extensions/cli/src/services/mcpTransports.ts:77-90](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/services/mcpTransports.ts#L77-L90); searched `rg -n -i "sandbox|seccomp|landlock|seatbelt|bwrap|firejail|sandbox-exec"` in `extensions/cli/src` → 0 hits (No sandbox or OS isolation primitive anywhere in the CLI.) (verified)
  - *To reach the next level:* Not even the main exec tool is sandboxed.
- **D L0:** No sandbox exists to be on by default. — searched `rg -n -i "sandbox|seccomp|landlock|seatbelt|bwrap|firejail|sandbox-exec"` in `extensions/cli/src` → 0 hits (No sandbox or OS isolation primitive anywhere in the CLI.) (verified)
  - *To reach the next level:* No sandbox on by default.
- **B L0:** Host-equivalent: home directory, ~/.ssh and ~/.aws, credentials in the environment, and unrestricted network are all reachable from executed commands. — [extensions/cli/src/tools/runTerminalCommand.ts:192](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L192); [extensions/cli/src/services/mcpTransports.ts:77-90](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/services/mcpTransports.ts#L77-L90) (verified)
  - *To reach the next level:* Execution is not confined to the workspace, and secrets and network are not withheld.
- **Cap:** none

### C5 Untrusted input blast radius — 0.25 (high)

Fetched web pages, file contents, MCP results and repository instruction files all enter the model's context with the same standing as the user's request; nothing tracks where content came from. The general approval prompt still stands in front of shell, writes and MCP calls, but it is not tied to whether untrusted content was read, and Read and Fetch run unprompted, which is enough to read a file and send it to an attacker's URL. The approval gate also does not cover every path, so a hijacked session can both exfiltrate and take irreversible action with no human involved.

- **S L2:** Shell, write and MCP calls require approval regardless of what has been read, but egress (Fetch) does not, and nothing reacts to untrusted content specifically. — [extensions/cli/src/permissions/defaultPolicies.ts:34-37](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/defaultPolicies.ts#L34-L37); [extensions/cli/src/permissions/defaultPolicies.ts:21](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/defaultPolicies.ts#L21); searched `rg -n -i "untrusted|taint|provenance|prompt.injection"` in `extensions/cli/src` → 0 hits (No provenance or taint tracking for tool results or fetched content.) (verified)
  - *To reach the next level:* No Rule-of-Two enforcement: egress and state change are not forced through approval once untrusted content is in context.
- **C L0:** Untrusted sources are not distinguished; tool results, fetched pages and MCP output are added to history like any other message. — searched `rg -n -i "untrusted|taint|provenance|prompt.injection"` in `extensions/cli/src` → 0 hits (No provenance or taint tracking for tool results or fetched content.); [extensions/cli/src/services/ChatHistoryService.ts:297-300](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/services/ChatHistoryService.ts#L297-L300) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished from principal input.
- **D L2:** The approval gate is on by default but the operator can drop it silently with --auto or permissions.yaml allow rules. — [extensions/cli/src/shared-options.ts:15](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/shared-options.ts#L15); [extensions/cli/src/permissions/permissionsYamlLoader.ts:11-13](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/permissionsYamlLoader.ts#L11-L13) (verified)
  - *To reach the next level:* Disabling is not warned, and allow rules can widen silently.
- **B L0:** A hijacked session can read secrets via Read and exfiltrate via Fetch without approval; other paths also run without approval. — [extensions/cli/src/permissions/defaultPolicies.ts:23](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/defaultPolicies.ts#L23); [extensions/cli/src/permissions/defaultPolicies.ts:21](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/defaultPolicies.ts#L21); [extensions/cli/src/permissions/defaultPolicies.ts:24](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/defaultPolicies.ts#L24) (verified)
  - *To reach the next level:* Irreversible action and exfiltration are not both behind human approval.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.25 (high)

The CLI has no long-term memory store, and its permissions and main config come only from the user's home directory or explicit flags. But it silently loads AGENTS.md/CLAUDE.md and .continue/rules from the working directory into the system prompt, and repository content can influence execution without any trust prompt. Project-level hook files are parsed but never executed at this commit. Saved sessions live in ~/.continue/sessions and are only reused on explicit --resume.

- **S L1:** Instruction files from the workspace load silently as system-prompt content, and other workspace content can influence execution. — [extensions/cli/src/systemMessage.ts:167-177](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/systemMessage.ts#L167-L177); [extensions/cli/src/systemMessage.ts:90-93](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/systemMessage.ts#L90-L93); searched `rg -n "firePreToolUse|fireSessionStart|fireUserPromptSubmit"` in `extensions/cli/src` → 3 hits (All three hits are the function definitions in hooks/fireHook.ts; nothing calls them, so project hooks are loaded but never executed at this commit.) (verified)
  - *To reach the next level:* Project files can still influence execution without an explicit workspace-trust decision.
- **C L1:** Permissions (user-scope permissions.yaml) and config (~/.continue/config.yaml) are protected from the workspace, but instruction files and rules are not. — [extensions/cli/src/permissions/permissionsYamlLoader.ts:11-13](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/permissionsYamlLoader.ts#L11-L13); [extensions/cli/src/configLoader.ts:95-99](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/configLoader.ts#L95-L99); [extensions/cli/src/systemMessage.ts:167-177](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/systemMessage.ts#L167-L177) (verified)
  - *To reach the next level:* Auto-loaded workspace files and rules are uncontrolled.
- **D L2:** Local, single-user storage under ~/.continue; sessions are per-session JSON files. — [core/util/history.ts:131-134](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/core/util/history.ts#L131-L134) (verified)
  - *To reach the next level:* No guard preventing the agent's own tools from writing other sessions' files.
- **B L1:** Poisoned instruction files persist in the repository across the user's sessions and can steer tool use, including unapproved Search/Fetch/Read. — [extensions/cli/src/systemMessage.ts:167-177](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/systemMessage.ts#L167-L177); [extensions/cli/src/permissions/defaultPolicies.ts:24](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/defaultPolicies.ts#L24) (verified)
  - *To reach the next level:* Persistent poisoned context can still trigger unapproved tool use.
- **Cap:** C6-REPOCONFIG — Repository content can influence execution without any user trust decision.

### C7 Third-party extensions — 0.23 (high)

Third-party code enters as MCP servers the user adds to ~/.continue/config.yaml or passes with --mcp; the CLI does not load MCP configuration from the workspace. Server commands run exactly as configured with no version pinning or integrity check, and stdio servers are launched with the user's full environment merged in. MCP tool calls go through the approval prompt by default.

- **S L1:** MCP servers are user-chosen and launched as configured, with no pinning or hash verification. — [extensions/cli/src/services/mcpTransports.ts:77-90](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/services/mcpTransports.ts#L77-L90); searched `rg -n -i "integrity|sha256|checksum|signature"` in `extensions/cli/src/services/MCPService.ts extensions/cli/src/services/mcpTransports.ts` → 0 hits (No pinning or integrity verification of MCP server packages.) (verified)
  - *To reach the next level:* No version pinning or integrity check for MCP servers.
- **C L0:** No extension type is verified. — searched `rg -n -i "integrity|sha256|checksum|signature"` in `extensions/cli/src/services/MCPService.ts extensions/cli/src/services/mcpTransports.ts` → 0 hits (No pinning or integrity verification of MCP server packages.) (verified)
  - *To reach the next level:* No extension type is verified.
- **D L2:** Nothing third-party is enabled from the workspace; servers come from user-scope config or explicit flags, without a confirmation showing what will run. — [extensions/cli/src/configLoader.ts:95-99](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/configLoader.ts#L95-L99) (verified)
  - *To reach the next level:* Adding a server doesn't show the exact package/command and permissions for confirmation.
- **B L1:** Stdio MCP servers run as separate processes, same user, with the full process environment. — [extensions/cli/src/services/mcpTransports.ts:77-90](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/services/mcpTransports.ts#L77-L90) (verified)
  - *To reach the next level:* Extension processes are not given a scrubbed environment.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.25 (high)

Read and Edit refuse a built-in list of secret-looking files (.env, keys, .aws/, .ssh/); the matching does not cover every path, and the check does not apply to Bash. Subprocesses inherit the full environment. Telemetry is metrics-only and off unless OpenTelemetry is configured, and logs default to info level, but session transcripts with full tool output are stored in plaintext in ~/.continue/sessions and there is no redaction anywhere.

- **S L1:** Secrets come from env/config; the only protection is a filename denylist on Read/Edit. — [extensions/cli/src/tools/readFile.ts:56](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/readFile.ts#L56); [core/indexing/ignore.ts:246-254](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/core/indexing/ignore.ts#L246-L254); searched `rg -n -i "redact|mask" --glob '!*.test.*'` in `extensions/cli/src` → 0 hits (No redaction or masking helper for logs, transcripts or model-bound content.) (verified)
  - *To reach the next level:* No type-level masking or log/transcript redaction.
- **C L1:** Only the model-bound Read/Edit path has a (denylist) filter; logs, transcripts and subprocess environments are unprotected. — [extensions/cli/src/tools/runTerminalCommand.ts:192](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L192); [core/util/history.ts:131-134](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/core/util/history.ts#L131-L134); searched `rg -n -i "redact|mask" --glob '!*.test.*'` in `extensions/cli/src` → 0 hits (No redaction or masking helper for logs, transcripts or model-bound content.) (verified)
  - *To reach the next level:* Logs and transcripts are not protected.
- **D L2:** Telemetry is opt-in (requires OTEL configuration) and metrics-only; logging defaults to info with tool arguments at debug. — [extensions/cli/src/telemetry/telemetryService.ts:84](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/telemetry/telemetryService.ts#L84); [extensions/cli/src/util/logger.ts:103](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/util/logger.ts#L103) (verified)
  - *To reach the next level:* No always-on redaction.
- **B L0:** Long-lived keys in the user's environment and config are reachable by every subprocess and by the model via Bash. — [extensions/cli/src/tools/runTerminalCommand.ts:192](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L192); [extensions/cli/src/services/mcpTransports.ts:77-90](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/services/mcpTransports.ts#L77-L90) (verified)
  - *To reach the next level:* Keys reachable by subprocesses are long-lived and unscoped.
- **Cap:** none

### C9 Audit & traceability — 0.45 (high)

Each session is saved as a structured JSON transcript in ~/.continue/sessions that records tool calls, their arguments, results and status, including denials. It is written by the agent process itself in a location its own tools can modify, carries no actor attribution or tamper evidence, and save failures are only logged. Sub-agent runs temporarily disable the history service, so their tool calls are not recorded individually.

- **S L2:** A structured local session transcript records tool calls with arguments, results and status. — [extensions/cli/src/services/ChatHistoryService.ts:297-300](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/services/ChatHistoryService.ts#L297-L300); [extensions/cli/src/session.ts:279-295](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/session.ts#L279-L295); [core/util/history.ts:131-134](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/core/util/history.ts#L131-L134) (verified)
  - *To reach the next level:* No actor attribution (approver, principal) or correlation IDs across sub-agents.
- **C L2:** Built-in and MCP tool calls and user denials are recorded; sub-agent tool calls are not. — [extensions/cli/src/stream/streamChatResponse.helpers.ts:520-526](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/stream/streamChatResponse.helpers.ts#L520-L526); [extensions/cli/src/subagent/executor.ts:78-89](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/subagent/executor.ts#L78-L89) (verified)
  - *To reach the next level:* Sub-agent tool calls and explicit approvals are not recorded.
- **D L2:** On by default, stored outside the workspace in ~/.continue/sessions, but writable by the agent process and its tools. — [core/util/history.ts:131-134](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/core/util/history.ts#L131-L134) (verified)
  - *To reach the next level:* Record is written by a component the model's tools can alter.
- **B L1:** Saves are best-effort; failures are caught and only logged while actions proceed. — [extensions/cli/src/session.ts:279-295](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/session.ts#L279-L295) (verified)
  - *To reach the next level:* Errors are not surfaced and records are not guaranteed durable per action.
- **Cap:** none

### C10 Limits & kill switch — 0.20 (high)

The agent loop is an unbounded while(true) with no step, wall-clock or cost limit. Shell commands have an idle timeout of 180 seconds that resets whenever output appears, and the model can raise it to 600 seconds per call; background jobs are capped at five. Pressing Escape aborts the model stream cooperatively, but a running shell command is not tied to that abort signal.

- **S L1:** Only a per-command idle timeout exists; no iteration, wall-clock or cost cap on the loop. — [extensions/cli/src/stream/streamChatResponse.ts:443](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/stream/streamChatResponse.ts#L443); [extensions/cli/src/tools/runTerminalCommand.ts:199-203](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L199-L203); searched `rg -n -i "maxTurns|max_turns|maxIterations|max_iterations|maxSteps|costLimit|budget"` in `extensions/cli/src` → 2 hits (Both hits are in services/circular-dependencies.test.ts (a test helper), not an agent-loop limit.) (verified)
  - *To reach the next level:* No iteration cap combined with a wall-clock or token/cost cap.
- **C L1:** The only limit applies to individual shell commands (plus a five-job background cap). — [extensions/cli/src/tools/runTerminalCommand.ts:199-203](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L199-L203); [extensions/cli/src/services/BackgroundJobService.ts:23](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/services/BackgroundJobService.ts#L23) (verified)
  - *To reach the next level:* No limit on the top-level loop.
- **D L1:** The model can raise the per-command timeout up to 600s, and the timeout resets on any output. — [extensions/cli/src/tools/runTerminalCommand.ts:199-203](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/tools/runTerminalCommand.ts#L199-L203) (verified)
  - *To reach the next level:* Defaults can be raised by the model.
- **B L0:** No ceiling: auto-approved tools (Read, Fetch, Search) can loop and spend indefinitely. — [extensions/cli/src/stream/streamChatResponse.ts:443](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/stream/streamChatResponse.ts#L443); [extensions/cli/src/permissions/defaultPolicies.ts:21](https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/permissions/defaultPolicies.ts#L21) (verified)
  - *To reach the next level:* No ceiling on runaway loops or spend.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Fetch of any URL (extensions/cli/src/tools/fetch.ts:59), repo files via Read, MCP results, AGENTS.md (extensions/cli/src/systemMessage.ts:167) · [B] sensitive data/systems: Any file readable by the user via Read (extensions/cli/src/tools/readFile.ts:83) and the inherited environment (extensions/cli/src/tools/runTerminalCommand.ts:192) · [C] state change / egress: Auto-approved Fetch egress (extensions/cli/src/permissions/defaultPolicies.ts:21) and other paths that run without approval · Same default session? Yes

## Highest-impact improvements
1. Ensure every tool that reaches a subprocess is either gated by approval or invokes it without a shell. — C2 C L1→L2, +0.075 before caps (Playbook 3)
2. Make Fetch 'ask' by default and block loopback, private and link-local addresses (rechecking redirects). — C5 B L0→L1, +0.050 before caps (Playbook 1)
3. Make 'don't ask again' save the exact command rather than a first-word prefix wildcard, and harden the Bash approval display. — C2 S L2→L3, +0.075 before caps (Playbook 5)
4. Pass a scrubbed environment (allowlisted variables only) to Bash and MCP stdio subprocesses. — C8 C L1→L2, +0.075 before caps (Playbook 4)
5. Add default iteration and session cost caps to the agent loop, enforced in code. — C10 S L1→L2, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was installed, built, run or tested, and all findings were established by reading the code, not by executing it.
- Only the CLI (extensions/cli) in interactive mode was scored. Headless -p mode auto-allows Bash and all MCP tools by default and would score lower. The VS Code/JetBrains extensions (core/) use a different policy set (e.g. Fetch requires permission there, and terminal-security can auto-approve 'safe' commands) and were not scored.
- The remote default configuration (continuedev/default-cli-config fetched from the Continue hub when no ~/.continue/config.yaml exists) was not inspected; its contents are outside the repository.
- Hooks loaded from project .continue/.claude settings files are not executed at this commit (no call sites for the fire* functions); a later commit that wires them up would create a repo-controlled code-execution path.
- The README states the repository is no longer actively maintained and is read-only, so the findings are unlikely to be fixed upstream.
- No reviewer-steering text aimed at AI auditors was found in the repository.
