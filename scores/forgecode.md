# Defense-in-depth score: Forge (forgecode)

**Repo:** https://github.com/tailcallhq/forgecode · **Commit:** `8eb3e243307b2a84e882f919a1ce1082715ba63f` · **Reviewed:** 2026-10-05
**What it is:** Terminal coding agent CLI (Rust) with interactive TUI, one-shot mode and a zsh plugin, supporting many model providers, MCP servers, custom agents and skills.
**Category:** Coding
**Scored configuration:** Interactive `forge` CLI with no flags on a fresh install, using the embedded defaults in crates/forge_config/.forge.toml (restricted mode off, sub-agents on, telemetry on in release builds).
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions opt-in · sub agents yes · external communication yes

## Score: 1.9 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C2 | Approval gates | L2 | L2 | L1 | L1 | 0.40 | G2 | **0.25** (alt) | High |
| C3 | Tool & action scoping | L0 | L1 | L1 | L0 | 0.12 | none | **0.12** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L1 | L1 | L1 | 0.17 | C6-REPOCONFIG | **0.17** | High |
| C7 | Third-party extensions | L1 | L1 | L1 | L1 | 0.25 | G2 | **0.25** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L0 | L0 | 0.15 | none | **0.15** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | none | **0.50** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | none | **0.40** | High |


As shipped, Forge runs every shell command, file change and web request the model chooses on your machine with your full permissions and environment, without asking first. There is no sandbox, and the opt-in restricted mode allows everything until you write your own rules. Content read from the web, the repository or MCP servers can therefore steer it to leak credentials and make irreversible changes, and opening an untrusted repository can change its configuration and agent behaviour. Telemetry in release builds also sends prompt text to a third party by default.

## Critical gaps
- Shell commands run on the host as the user with the full inherited environment, so any credential the user holds is in reach. (ASI03, T3, LLM06; C1). Evidence: [crates/forge_infra/src/executor.rs:35-44](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/executor.rs#L35-L44)
- The permission rules for the opt-in approval mode can be rewritten by the agent's own tools and are re-read on every check. (ASI09, ASI02, T10; C2). Evidence: [crates/forge_services/src/policy.rs:135-146](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/policy.rs#L135-L146)
- There is no execution sandbox; model-written commands and MCP servers run directly on the host. (ASI05, T11, LLM05; C4). Evidence: [crates/forge_infra/src/executor.rs:57-63](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/executor.rs#L57-L63)
- A hijacked session can exfiltrate data with fetch or the shell and take irreversible actions, all without a human. (ASI01, T6, LLM01; C5). Evidence: [crates/forge_services/src/tool_services/fetch.rs:64-72](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/tool_services/fetch.rs#L64-L72); [crates/forge_config/.forge.toml:22](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_config/.forge.toml#L22)
- Workspace `.env` files, AGENTS.md and `.forge/agents` load without a trust decision and can change configuration and replace the default agent. (ASI06, ASI04, T1; C6). Evidence: [crates/forge_config/src/reader.rs:13-31](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_config/src/reader.rs#L13-L31); [crates/forge_repo/src/agent.rs:63-70](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_repo/src/agent.rs#L63-L70)
- The project-scope MCP trust decision does not hold on every path in the default configuration. (ASI04, T17, LLM03; C7). Evidence: [crates/forge_services/src/mcp/manager.rs:116-121](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/mcp/manager.rs#L116-L121)

## Criterion details

### C1 Identity & least privilege: 0.00 (high confidence)

Forge runs as the local user and does nothing to narrow that authority. Shell commands start from the user's own shell with the full process environment, so cloud keys, Git tokens and provider API keys present there are available to anything the model runs. There is no separate identity, scoped token or authorization layer in code. If the agent is steered into misuse, it can do whatever the user's account can do.

- **S L0:** The shell tool spawns the user's $SHELL with the parent environment inherited unchanged; there is no dedicated or scoped identity. Evidence: [crates/forge_infra/src/executor.rs:35-44](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/executor.rs#L35-L44); [crates/forge_infra/src/env.rs:20-24](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/env.rs#L20-L24) (verified)
  - *To reach the next level:* No dedicated identity or scoped credentials; even a role-scoped default would need environment scrubbing or narrowed tokens.
- **C L0:** No authorization check sits on any tool path; the executor only adds extra variables on top of the inherited environment. Evidence: [crates/forge_infra/src/executor.rs:78-88](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/executor.rs#L78-L88); searched `rg -n -e 'env_clear' --type rust` in `crates` → 0 hits (no subprocess spawn clears the inherited environment) (verified)
  - *To reach the next level:* No path passes through an authorization layer, not even the main shell tool.
- **D L0:** The default install runs every tool with the user's full ambient authority. Evidence: [crates/forge_config/.forge.toml:22](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_config/.forge.toml#L22) (verified)
  - *To reach the next level:* No narrower default identity or read-only default exists.
- **B L0:** A hijacked session reaches everything the user's account and inherited credentials can reach, across services. Evidence: [crates/forge_infra/src/executor.rs:57-63](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/executor.rs#L57-L63) (verified)
  - *To reach the next level:* Nothing limits the reachable systems to one project or to read-only access.
- **Cap:** none

### C2 Approval gates: 0.25 (high confidence)

Out of the box Forge asks for no approval at all: shell commands, file writes and deletions, and web fetches run as soon as the model calls them. An opt-in restricted mode can ask the user before built-in tools run, but even when switched on, the shipped permission rules allow every read, write, command and URL, so the user also has to write their own rules. Commands are matched as plain text patterns, MCP tools and sub-agent calls never pass through the check, and the rules file can be rewritten by the agent's own tools. File edits made through the file tools can be undone; shell commands cannot.

- **default configuration** (default; raw 0.05 → 0.05)
  - **S L0:** In the default configuration no tool call is put in front of a human; restricted mode is off. Evidence: [crates/forge_config/.forge.toml:22](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_config/.forge.toml#L22) (verified)
    - *To reach the next level:* No per-call approval exists by default.
  - **C L0:** The most powerful tool, the shell, runs without any gate by default. Evidence: [crates/forge_infra/src/executor.rs:57-63](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/executor.rs#L57-L63) (verified)
    - *To reach the next level:* The shell and other mutating tools would need to pass through a gate.
  - **D L0:** Approval is opt-in through the restricted setting, and the shipped policy file allows everything even then. Evidence: [crates/forge_config/.forge.toml:22](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_config/.forge.toml#L22); [crates/forge_services/src/permissions.default.yaml:1-13](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/permissions.default.yaml#L1-L13) (verified)
    - *To reach the next level:* Approval would need to be on by default.
  - **B L1:** File-tool writes are snapshotted for undo, but shell commands (deletes, pushes, network calls) are irreversible. Evidence: [crates/forge_services/src/tool_services/fs_write.rs:99-101](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/tool_services/fs_write.rs#L99-L101) (verified)
    - *To reach the next level:* Shell side effects have no checkpoint or rollback, so the common case is not reversible.
- **opt-in restricted mode with user-written permission rules** (alt; raw 0.40, cap G2 → 0.25) ← counted
  - **S L2:** When restricted mode is on and a rule says confirm, the user gets Accept, Reject or Accept-and-Remember per call, with glob rules matched on the raw command string. Evidence: [crates/forge_services/src/policy.rs:173-203](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/policy.rs#L173-L203); [crates/forge_domain/src/policies/rule.rs:79-86](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_domain/src/policies/rule.rs#L79-L86); [crates/forge_services/src/policy.rs:251-262](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/policy.rs#L251-L262) (verified)
    - *To reach the next level:* Rules match unparsed strings and remembered approvals widen to a command-plus-subcommand prefix, so the approver is not guaranteed the policy reflects the exact call.
  - **C L2:** Built-in read, write, patch, remove, shell and fetch map to policy operations, while undo, task delegation, agent tools and MCP tools are never checked. Evidence: [crates/forge_domain/src/tools/catalog.rs:1007-1015](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_domain/src/tools/catalog.rs#L1007-L1015); [crates/forge_app/src/tool_registry.rs:189-192](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_app/src/tool_registry.rs#L189-L192) (verified)
    - *To reach the next level:* MCP tools and delegation would need to traverse the same gate.
  - **D L1:** The mode is off by default, the shipped rules allow everything when it is on, and the rules file is re-read on every check from a user-scope path that the agent's own write and shell tools can reach. Evidence: [crates/forge_config/.forge.toml:22](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_config/.forge.toml#L22); [crates/forge_services/src/permissions.default.yaml:1-13](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/permissions.default.yaml#L1-L13); [crates/forge_services/src/policy.rs:135-146](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/policy.rs#L135-L146) (verified)
    - *To reach the next level:* Neither the restricted setting nor the permission rules are protected from the agent's own tools.
  - **B L1:** Wrongly approved shell commands are irreversible; only file-tool edits have undo snapshots. Evidence: [crates/forge_services/src/tool_services/fs_write.rs:99-101](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/tool_services/fs_write.rs#L99-L101) (verified)
    - *To reach the next level:* No rollback for shell side effects and no rate limit on approvals.
- **Cap:** G2: The permission rules are re-read on every check from a file the agent's own write and shell tools can modify, so the model can loosen the opt-in gate at runtime.

### C3 Tool & action scoping: 0.12 (high confidence)

Forge's tools are general purpose: the shell tool takes any command string, the fetch tool takes any URL, and the file tools accept any absolute path on the machine. The only argument checks are that paths are absolute and commands are not empty; there is no workspace containment, host allowlist or block on internal addresses. Each agent has a tool list, but the default agent gets shell, write, remove, fetch, delegation and all MCP tools. A misused tool can therefore act on the whole machine and any reachable host.

- **S L0:** Shell commands, URLs and file paths are passed through without allowlist validation; file tools only require the path to be absolute. Evidence: [crates/forge_services/src/utils/path.rs:13-19](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/utils/path.rs#L13-L19); [crates/forge_services/src/tool_services/shell.rs:33-38](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/tool_services/shell.rs#L33-L38); [crates/forge_services/src/tool_services/fetch.rs:64-72](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/tool_services/fetch.rs#L64-L72) (verified)
  - *To reach the next level:* No allowlist validation such as resolved-path containment in the workspace or a host allowlist that blocks internal addresses.
- **C L1:** A few built-in tools apply basic shape checks (absolute path, non-empty command); fetch and MCP tools have none. Evidence: searched `rg -n -e 'assert_absolute_path\(' --type rust` in `crates/forge_services/src/tool_services` → 7 hits (call sites in write, image_read, read, remove, undo and patch (two); a shape check, not containment); searched `rg -n -i -e '169\.254|ssrf|allowed_hosts|is_private|is_loopback' --type rust` in `crates/forge_services crates/forge_app` → 0 hits (verified)
  - *To reach the next level:* Most built-in tools would need real validation, not shape checks.
- **D L1:** Tools are assigned per agent and enforced against the agent's list, but the default forge agent receives shell, write, remove, fetch, task and all MCP tools. Evidence: [crates/forge_app/src/tool_registry.rs:361-364](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_app/src/tool_registry.rs#L361-L364); [crates/forge_repo/src/agents/forge.md:7-22](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_repo/src/agents/forge.md#L7-L22) (verified)
  - *To reach the next level:* Write and exec tools are in the default set; a read-only default would be needed.
- **B L0:** The shell and file tools reach the whole machine with the user's permissions and fetch reaches any host. Evidence: [crates/forge_infra/src/executor.rs:57-63](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/executor.rs#L57-L63) (verified)
  - *To reach the next level:* Tools are not scoped to the workspace or to a host set.
- **Cap:** none

### C4 Code-execution isolation: 0.00 (high confidence)

Every shell command the model writes runs directly on the host as the user, through the user's own shell, with the full environment. MCP stdio servers are also launched as ordinary host processes. The `--sandbox` flag only creates a separate git worktree and branch; it is not an isolation boundary. Anything the model runs, including repository scripts, can read and change everything the user can, and use every credential in the environment.

- **S L0:** Commands run as a same-user subprocess of $SHELL -c with no isolation primitive. Evidence: [crates/forge_infra/src/executor.rs:35-63](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/executor.rs#L35-L63); searched `rg -n -i -e 'landlock|seatbelt|sandbox-exec|bwrap|bubblewrap|seccomp|firejail|nsjail|docker run|gvisor|firecracker' --type rust` in `crates` → 2 hits (both hits are test strings in terminal_context.rs; no isolation backend exists) (verified)
  - *To reach the next level:* No OS-level separation such as a container, low-privilege user or OS sandbox profile.
- **C L0:** No execution path is sandboxed: shell commands and MCP stdio servers both run on the host. Evidence: [crates/forge_infra/src/mcp_client.rs:99-105](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/mcp_client.rs#L99-L105) (verified)
  - *To reach the next level:* The main exec tool would need to run inside a sandbox.
- **D L0:** There is no sandbox to enable; the `--sandbox` option creates a git worktree only. Evidence: [crates/forge_main/src/cli.rs:49-51](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_main/src/cli.rs#L49-L51); [crates/forge_main/src/sandbox.rs:19-20](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_main/src/sandbox.rs#L19-L20) (verified)
  - *To reach the next level:* A sandbox would need to exist and be on by default.
- **B L0:** Executed code has host-equivalent reach: the user's home directory, all files, unrestricted network and the inherited credentials. Evidence: [crates/forge_infra/src/executor.rs:78-88](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/executor.rs#L78-L88) (verified)
  - *To reach the next level:* Execution would need to be confined to the workspace without secrets in the environment.
- **Cap:** none

### C5 Untrusted input blast radius: 0.00 (high confidence)

Forge reads untrusted content from the web (fetch tool), repository files, command output and MCP tool results, and puts it into the model's context with no marking or special handling. In the same session it holds the user's credentials and can run shell commands, write files and make arbitrary web requests without asking. If injected text takes over the session, it can both send data out and make irreversible changes with no human involved.

- **S L0:** Nothing in code limits what the model can do after reading untrusted content; there is no detection, tagging or gating tied to provenance. Evidence: searched `rg -n -i -e 'prompt.injection|untrusted|taint|quarantin|provenance' --type rust` in `crates` → 3 hits (all three hits are the project-local MCP config trust prompt, not handling of content read during a session) (verified)
  - *To reach the next level:* Egress and state-changing tools would need human approval once untrusted content enters the session.
- **C L0:** Fetched pages, file contents, shell output and MCP results all enter context as ordinary tool results. Evidence: [crates/forge_app/src/tool_registry.rs:189-210](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_app/src/tool_registry.rs#L189-L210); [crates/forge_services/src/tool_services/fetch.rs:125-127](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/tool_services/fetch.rs#L125-L127) (verified)
  - *To reach the next level:* Untrusted sources would need to be distinguished from the principal's instructions.
- **D L0:** No containment exists to be on by default; restricted mode is off. Evidence: [crates/forge_config/.forge.toml:22](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_config/.forge.toml#L22) (verified)
  - *To reach the next level:* Some provenance-based limit would need to exist and be on by default.
- **B L0:** A hijacked session can read secrets and the user's files, send them out with fetch or the shell, and delete or push changes, all unattended. Evidence: [crates/forge_services/src/tool_services/fetch.rs:64-72](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/tool_services/fetch.rs#L64-L72); [crates/forge_infra/src/executor.rs:57-63](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/executor.rs#L57-L63) (verified)
  - *To reach the next level:* Exfiltration or irreversible actions would need a human in the loop.
- **Cap:** C5-WORSTCASE: In the default configuration a hijacked session can exfiltrate data and take irreversible actions with no human involved.

### C6 Memory, context & configuration integrity: 0.17 (high confidence)

Forge loads several things from the project it is opened in without asking: `.env` files from the working directory and every parent directory (their FORGE_ settings and provider variables feed the configuration), AGENTS.md instructions, and custom agents, skills and commands under `.forge/`, where a project agent can replace the built-in default agent with its own tools and prompt. Only a project `.mcp.json` triggers a trust prompt, remembered per file content. The model can also write AGENTS.md, so injected instructions can persist into later sessions. Opening an untrusted repository can therefore change endpoints, settings and the agent's behaviour before the user does anything.

- **S L0:** Repository files can define or override agents (including the default agent's tool list and prompt) and set configuration through `.env` with no prompt. Evidence: [crates/forge_config/src/reader.rs:13-31](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_config/src/reader.rs#L13-L31); [crates/forge_repo/src/agent.rs:63-70](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_repo/src/agent.rs#L63-L70) (verified)
  - *To reach the next level:* Security-relevant project configuration would need an explicit workspace-trust decision.
- **C L1:** Only the project MCP config passes a trust prompt; `.env`, AGENTS.md, agents, skills and commands load silently. Evidence: [crates/forge_services/src/mcp/manager.rs:74-113](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/mcp/manager.rs#L74-L113); [crates/forge_services/src/instructions.rs:32-44](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/instructions.rs#L32-L44) (verified)
  - *To reach the next level:* All auto-loaded files and settings would need to be covered by the trust decision.
- **D L1:** State is local to the user (conversations keyed by ID in the user's own database), but with no control over what loads, isolation earns little: the model can write files that later sessions auto-load. Evidence: [crates/forge_domain/src/env.rs:113-114](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_domain/src/env.rs#L113-L114); [crates/forge_app/src/agent_executor.rs:58-64](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_app/src/agent_executor.rs#L58-L64) (verified)
  - *To reach the next level:* The model would need to be unable to write into auto-loaded context, and loads would need a controlled path before isolation counts.
- **B L1:** Instructions written into AGENTS.md or `.forge/agents` persist across the user's sessions and can drive tool use. Evidence: [crates/forge_services/src/instructions.rs:68-77](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/instructions.rs#L68-L77) (verified)
  - *To reach the next level:* Persisted instructions would need to be limited to text output or gated actions.
- **Cap:** C6-REPOCONFIG: Workspace files load without a trust decision and can set configuration, including endpoints, and replace the default agent's tools and prompt.

### C7 Third-party extensions: 0.25 (high confidence)

MCP servers are the main third-party extension. The user adds them by command line or URL in a user-scope file or a project `.mcp.json`; nothing is pinned or hash-checked, so a command like `npx some-server` runs whatever is current. A project `.mcp.json` triggers a trust prompt that is remembered per file content and asks again when the file changes, but the prompt shows only the file path, not the commands it will run, and the trust decision does not hold in every configuration. Launched servers are ordinary host processes that inherit the user's full environment.

- **S L1:** Servers come from user-chosen commands or URLs with no version pinning or integrity check. Evidence: [crates/forge_infra/src/mcp_client.rs:99-105](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/mcp_client.rs#L99-L105); searched `rg -n -i -e 'sha256|checksum|signature|sigstore' --type rust` in `crates/forge_services/src/mcp crates/forge_infra/src/mcp_client.rs` → 0 hits (verified)
  - *To reach the next level:* Server packages or binaries would need pinned versions.
- **C L1:** Only the project MCP config passes a consent check (keyed by a hash of its content); user-scope servers, skills and agents are not verified. Evidence: [crates/forge_services/src/mcp/manager.rs:83-94](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/mcp/manager.rs#L83-L94) (verified)
  - *To reach the next level:* Most extension types would need verification.
- **D L1:** A project MCP config is enabled after a generic prompt that names only the file path, and the trust decision does not hold on every path in the default configuration. Evidence: [crates/forge_services/src/mcp/manager.rs:116-121](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_services/src/mcp/manager.rs#L116-L121) (verified)
  - *To reach the next level:* The consent prompt would need to show the exact commands and URLs, and the decision would need to hold for every server the project file defines.
- **B L1:** Stdio servers run as separate host processes of the same user and inherit the full environment, plus any variables from their config. Evidence: [crates/forge_infra/src/mcp_client.rs:99-105](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/mcp_client.rs#L99-L105) (verified)
  - *To reach the next level:* Extension processes would need a scrubbed environment containing only their own configuration.
- **Cap:** G2: Repository-controlled configuration can get past the project-scope MCP trust decision, so the consent control does not hold at runtime.

### C8 Secrets & sensitive-data protection: 0.15 (high confidence)

Provider API keys are stored in a plaintext JSON file in the Forge config directory with owner-only permissions, and authorization headers are redacted in debug HTTP logs. Nothing else is masked: tool-call arguments are logged in full, shell commands inherit every secret in the environment, and the model can read the credentials file with its own tools. Release builds send usage telemetry to a third party by default, including the text of prompts, command-line arguments, the working directory and, on errors, the current conversation; the FORGE_TRACKER switch only removes the user-identifying extras.

- **S L1:** Credentials sit in a plaintext file set to 0600, and the only masking is header redaction on the debug HTTP log path; tool-call arguments are logged unredacted. Evidence: [crates/forge_repo/src/provider/provider_repo.rs:600-608](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_repo/src/provider/provider_repo.rs#L600-L608); [crates/forge_infra/src/http.rs:216-233](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/http.rs#L216-L233) (verified)
  - *To reach the next level:* No type-level masking or log filters on the main logging paths (tool-call logs), and no keychain or encryption at rest.
- **C L1:** Redaction covers only HTTP header debug logs; tool-call logs, telemetry, subprocess environments and saved conversations are not filtered. Evidence: [crates/forge_app/src/tool_registry.rs:102](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_app/src/tool_registry.rs#L102) (verified)
  - *To reach the next level:* Logs and transcripts would both need redaction.
- **D L0:** Telemetry is on by default in release builds and sends prompt text, arguments and conversations to a third-party analytics service. Evidence: [crates/forge_tracker/src/can_track.rs:12-15](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_tracker/src/can_track.rs#L12-L15); [crates/forge_tracker/src/dispatch.rs:119-134](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_tracker/src/dispatch.rs#L119-L134); [crates/forge_tracker/src/event.rs:97](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_tracker/src/event.rs#L97); [crates/forge_main/src/ui.rs:379](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_main/src/ui.rs#L379) (verified)
  - *To reach the next level:* Telemetry would need to be content-free, or opt-in.
- **B L0:** Long-lived provider keys and every credential in the user's environment are reachable by the model's shell and file tools and by every subprocess. Evidence: [crates/forge_domain/src/env.rs:172-174](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_domain/src/env.rs#L172-L174); [crates/forge_infra/src/executor.rs:35-44](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/executor.rs#L35-L44) (verified)
  - *To reach the next level:* Keys would need to be kept out of reach of model-driven tools and subprocesses.
- **Cap:** none

### C9 Audit & traceability: 0.50 (high confidence)

Every tool call, including MCP tools, is logged with its full arguments before it runs, and the conversation, including tool calls and results, is saved to a local database in the Forge config directory after each model request. Sub-agent runs are saved as their own conversations. The record does not say who approved what, sub-agent conversations are not linked to the parent run, and both the database and the log files sit where the agent's own shell and file tools can change or delete them.

- **S L2:** Tool calls are recorded with arguments in log files and in the persisted conversation, which holds calls and results. Evidence: [crates/forge_app/src/tool_registry.rs:102](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_app/src/tool_registry.rs#L102); [crates/forge_app/src/orch.rs:275-277](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_app/src/orch.rs#L275-L277) (verified)
  - *To reach the next level:* No attribution of approver or requesting principal and no correlation ID linking sub-agent runs to the parent.
- **C L2:** Built-in, MCP and sub-agent tool calls all pass through the same logging point, but approval decisions are not recorded as such. Evidence: [crates/forge_app/src/agent_executor.rs:66-76](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_app/src/agent_executor.rs#L66-L76) (verified)
  - *To reach the next level:* Approvals and denials, and the link from sub-agent runs to their parent, would need to be recorded.
- **D L2:** Recording is on by default and stored outside the workspace in the user's Forge directory, but the agent's own tools can reach that path. Evidence: [crates/forge_domain/src/env.rs:113-114](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_domain/src/env.rs#L113-L114) (verified)
  - *To reach the next level:* Records would need to be written by a component the model cannot control.
- **B L2:** The conversation is saved after each model request, so a crash loses at most the in-flight step. Evidence: [crates/forge_app/src/orch.rs:391-397](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_app/src/orch.rs#L391-L397) (verified)
  - *To reach the next level:* No guarantee of durable per-action records allowing exact replay, and nothing fails closed.
- **Cap:** none

### C10 Limits & kill switch: 0.40 (high confidence)

Each turn stops after 100 model requests by default, every built-in and MCP tool call is cut off after 300 seconds, and a turn ends after three failures of the same tool. A loop detector only adds a reminder to the model. There is no token or cost budget, sub-agents start their own fresh request budget with no limit on delegation depth or parallel tasks, and a project's own agent files can set a higher per-turn cap. Ctrl+C stops the current operation and the shell child is killed, but background processes the commands started keep running.

- **S L2:** A per-turn request cap and per-tool timeouts are enforced in code; the doom-loop detector is advisory. Evidence: [crates/forge_app/src/orch.rs:399-421](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_app/src/orch.rs#L399-L421); [crates/forge_app/src/tool_registry.rs:45-61](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_app/src/tool_registry.rs#L45-L61); [crates/forge_app/src/hooks/doom_loop.rs:229-245](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_app/src/hooks/doom_loop.rs#L229-L245) (verified)
  - *To reach the next level:* No token or cost cap and no breaker that stops repeated actions.
- **C L2:** Limits cover the top-level loop and tool calls, but delegated agents run a new chat with their own budget and are exempt from the tool timeout. Evidence: [crates/forge_app/src/tool_registry.rs:120-121](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_app/src/tool_registry.rs#L120-L121); [crates/forge_app/src/agent_executor.rs:79-85](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_app/src/agent_executor.rs#L79-L85) (verified)
  - *To reach the next level:* Sub-agents would need to count against the parent's budget.
- **D L1:** Defaults are sensible, but per-agent limits come from agent files, and a repository can supply agent files that set them. Evidence: [crates/forge_config/.forge.toml:12](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_config/.forge.toml#L12); [crates/forge_app/src/agent.rs:125-129](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_app/src/agent.rs#L125-L129) (verified)
  - *To reach the next level:* The model and workspace would need to be unable to raise the limits.
- **B L1:** With no spend ceiling and fresh budgets for each sub-agent, a runaway can spend and act for a long time; stopping kills only the direct shell child. Evidence: [crates/forge_infra/src/executor.rs:67](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_infra/src/executor.rs#L67); [crates/forge_main/src/ui.rs:380-386](https://github.com/tailcallhq/forgecode/blob/8eb3e243307b2a84e882f919a1ce1082715ba63f/crates/forge_main/src/ui.rs#L380-L386) (verified)
  - *To reach the next level:* Tight per-run time and cost ceilings and cleanup of processes left behind after a stop.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: fetch tool pages (crates/forge_services/src/tool_services/fetch.rs:64), repository files and MCP tool results (crates/forge_app/src/tool_registry.rs:189) · [B] sensitive data/systems: user's files, inherited environment and ~/.forge/.credentials.json (crates/forge_infra/src/executor.rs:38, crates/forge_domain/src/env.rs:173) · [C] state change / egress: shell, write/remove and fetch with no approval by default (crates/forge_config/.forge.toml:22) · Same default session? Yes

## Highest-impact improvements
1. Turn on per-call approval by default for shell, write/remove and fetch, with shipped rules that confirm rather than allow everything. (C2 D L0→L3, +0.150 before caps; Playbook 5)
2. Run shell commands in an OS sandbox (Landlock/Seatbelt or a hardened container) limited to the workspace, with network off by default. (C4 S L0→L3, +0.225 before caps; Playbook 3)
3. Require a workspace-trust decision before loading a project's .env, AGENTS.md, .forge/agents, skills and commands. (C6 S L0→L3, +0.225 before caps; Playbook 2)
4. Make telemetry opt-in or content-free: stop sending prompt text, arguments and conversations. (C8 D L0→L2, +0.100 before caps; Playbook 4)
5. Scrub the environment passed to shell commands and MCP servers down to what each needs. (C1 C L0→L2, +0.150 before caps; Playbook 4)

## Re-audit log
- C8 S: L2 → L1. Redaction covers only the debug HTTP header log; the main tool-call log records full arguments (crates/forge_app/src/tool_registry.rs:102), so masking exists on one path, not the main paths.
- C6 D: L2 → L1. D may be at most one level above S (L0); per-user locality earns little when repository files and model writes reach auto-loaded context.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The shallow clone carried no release tag; the workspace version is 0.1.0, while release builds get their version at build time, which is what turns telemetry on.
- Model-provider client code (crates/forge_repo/src/provider), the zsh plugin, the semantic-search workspace sync service and the VS Code extension were only skimmed.
- The self-update path (daily install-script download) is the project's own distribution channel and was not scored under third-party extensions.
- No text aimed at AI reviewers was found in the files read.
