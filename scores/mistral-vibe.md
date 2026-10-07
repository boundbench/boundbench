# Defense-in-Depth Score: Mistral Vibe

**Repo:** https://github.com/mistralai/mistral-vibe · **Commit:** `7c19608af06f6c61d63f8f7a5c3430da73fba2ab` (v2.25.8) · **Reviewed:** 2026-10-03
**What it is:** Minimal CLI coding agent by Mistral
**Category:** Coding
**Scored configuration:** Interactive `vibe` (Python TUI, legacy backend) with no flags, fresh install, default agent accept-edits, run in a project folder.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents yes · external communication no

## Score: 2.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L0 | L1 | 0.12 | — | **0.12** | High |
| C2 | Approval gates | L3 | L2 | L1 | L2 | 0.53 | G2 | **0.25** | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |
| C6 | Memory, context & configuration integrity | L1 | L2 | L2 | L1 | 0.38 | G2 | **0.25** | High |
| C7 | Third-party extensions | L1 | L0 | L0 | L0 | 0.07 | — | **0.07** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | G1 | **0.40** | High |


Vibe asks before running shell commands, fetching from new sites or calling MCP tools, and its command parser is unusually careful about hidden side effects. But the default profile writes and edits project files without asking, there is no sandbox, and unattended edits are not fully contained by the workspace-trust and approval controls. Avoid untrusted content in sessions that hold sensitive credentials.

## Critical gaps
- No execution isolation: shell commands, hooks and extensions run on the host as the user with the full environment and network. (ASI05, T11; C4) — [vibe/core/utils/shell.py:46-55](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/utils/shell.py#L46-L55); [vibe/core/utils/shell.py:58-59](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/utils/shell.py#L58-L59)
- Custom tool files are imported into the agent's own process, giving them its credentials and full environment. (ASI04, T17; C7) — [vibe/core/tools/manager.py:205-214](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/manager.py#L205-L214)

## Criterion details

### C1 Identity & least privilege — 0.12 (high)

Vibe runs as the local user with that user's full authority and does nothing to narrow it: every shell command inherits the complete process environment, including any provider API keys and cloud or Git credentials the user has exported. There is no per-tool identity or credential scoping; the only thing standing between a hijacked model and the user's accounts is the approval prompt on shell and network tools. The Mistral API key is kept in the OS keyring by default, which is good hygiene, but it is not a scoping control.

- **S L0:** Commands run as the OS user with the inherited process environment; no scoped or per-tool credentials exist. — [vibe/core/utils/shell.py:58-59](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/utils/shell.py#L58-L59); [vibe/core/utils/shell.py:46-55](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/utils/shell.py#L46-L55) (verified)
  - *To reach the next level:* Scrub the subprocess environment or issue tools narrower credentials than the user's ambient ones.
- **C L1:** Every bash subprocess receives the full os.environ copy; file tools act with the user's filesystem permissions. — [vibe/core/utils/shell.py:58-59](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/utils/shell.py#L58-L59); [vibe/core/hooks/executor.py:44-51](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/hooks/executor.py#L44-L51) (verified)
  - *To reach the next level:* No path uses a narrower identity than the user's; extensions and hooks also run as the user.
- **D L0:** The default install runs with the user's full ambient authority; nothing is narrowed by default. — [vibe/core/utils/shell.py:58-59](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/utils/shell.py#L58-L59) (verified)
  - *To reach the next level:* No narrower default identity or environment exists.
- **B L1:** A hijacked agent reaches whatever the user can (SSH keys, cloud CLIs, Git remotes); the per-call approval prompt on bash, web_fetch and MCP tools is the surviving independent layer. — [vibe/core/tools/builtins/bash.py:410-411](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/builtins/bash.py#L410-L411); [vibe/core/agent_loop/_loop.py:2973-2987](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/agent_loop/_loop.py#L2973-L2987) (verified)
  - *To reach the next level:* Blast radius would shrink only if credentials reachable from the agent were scoped to one system or short-lived.
- **Cap:** none

### C2 Approval gates — 0.25 (high)

Vibe has a careful approval system: shell commands are parsed with a real Bash grammar, compound commands are split and each part checked, dangerous options on otherwise read-only commands (find -exec, sort -o, git pager and config helpers) force a prompt, and the prompt shows the exact command. MCP tools, custom tools and sub-agent calls all pass through the same gate. However, the default agent profile auto-approves every file write and edit inside the project, so unattended edits are not fully contained by the approval and workspace-trust controls. An environment variable or config key also disables approval without a dedicated flag.

- **S L3:** Per-call approval shows the exact command; a tree-sitter Bash parse plus option guardrails (find -exec, git helpers) decide what needs a human, and reject is a first-class outcome. — [vibe/core/tools/builtins/bash.py:488-496](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/builtins/bash.py#L488-L496); [vibe/core/tools/builtins/_shell_permission_analysis.py:6-7](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/builtins/_shell_permission_analysis.py#L6-L7); [vibe/core/tools/builtins/_shell_command_policy.py:312-322](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/builtins/_shell_command_policy.py#L312-L322); [vibe/core/agent_loop/_loop.py:2940-2987](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/agent_loop/_loop.py#L2940-L2987) (verified)
  - *To reach the next level:* 'Always allow' grants generalise to arity-based wildcard patterns rather than staying bound to the exact approved arguments, so this falls short of full argument-level policy.
- **C L2:** All tools (built-in, MCP, custom, sub-agents) cross _should_execute_tool and unknown tools default to ASK, but the default profile auto-approves write_file/edit anywhere inside the workspace, which is not a read-only action. — [vibe/core/agents/models.py:88-99](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/agents/models.py#L88-L99); [vibe/core/tools/base.py:144](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/base.py#L144); [vibe/core/tools/utils.py:230-242](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/utils.py#L230-L242) (verified)
  - *To reach the next level:* Auto-approval in the default profile would need to be limited to a verified read-only set.
- **D L1:** Approval is on by default for shell and network, but it can be disabled through ordinary configuration (VIBE_BYPASS_TOOL_PERMISSIONS or a config key), and its default-on state is not tamper-resistant. — [vibe/core/config/layers/project.py:91-97](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/config/layers/project.py#L91-L97); [vibe/core/agents/models.py:88-99](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/agents/models.py#L88-L99); [vibe/core/config/layers/environment.py:13-17](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/config/layers/environment.py#L13-L17) (verified)
  - *To reach the next level:* Security settings that disable approval would need to be accepted only from user scope or an explicit loudly named flag.
- **B L2:** Edits by write_file/edit are snapshotted for rewind before running, but shell commands and network actions have no checkpoint, preview or rate limit. — [vibe/core/agent_loop/_loop.py:2839-2843](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/agent_loop/_loop.py#L2839-L2843) (verified)
  - *To reach the next level:* Checkpoints do not cover shell side effects and there are no rate limits on consequential actions or approval requests.
- **Cap:** G2 — The approval gate's default-on state is not tamper-resistant at runtime.

### C3 Tool & action scoping — 0.45 (high)

File tools resolve paths and check them against the workspace roots, prompt for anything outside, and treat .env files as sensitive; web_fetch approves per origin and refuses redirects to a different origin. But the central tool is a general shell that accepts any command string, there is no block on internal or metadata addresses, and the default tool set includes write, shell and network tools together. The model also chooses its own shell timeout with no upper bound.

- **S L2:** File tools use resolved-path containment and web_fetch re-checks redirect origins, but bash accepts an arbitrary shell string and web_fetch does not block localhost or metadata addresses. — [vibe/core/tools/utils.py:230-242](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/utils.py#L230-L242); [vibe/core/tools/builtins/web_fetch.py:244-246](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/builtins/web_fetch.py#L244-L246); [vibe/core/tools/builtins/bash.py:436-440](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/builtins/bash.py#L436-L440) (verified)
  - *To reach the next level:* General tools (raw shell, any URL) are not replaced by narrow ones, and URL validation lacks an internal-address block.
- **C L2:** Built-in tools validate via typed pydantic args and per-tool resolve_permission; MCP and custom tools get only their own JSON schema. — [vibe/core/tools/manager.py:471](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/manager.py#L471); [vibe/core/tools/mcp/tools.py:492-506](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/mcp/tools.py#L492-L506) (verified)
  - *To reach the next level:* No shared validation layer wraps MCP and custom Python tools.
- **D L2:** Agent profiles (plan, ask) and --enabled-tools/--disabled-tools let the operator select tool groups, but the default accept-edits profile ships bash, write_file, edit and web_fetch. — [vibe/core/config/vibe_schema.py:472-474](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/config/vibe_schema.py#L472-L474); [vibe/core/agents/models.py:88-99](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/agents/models.py#L88-L99) (verified)
  - *To reach the next level:* The default tool set is not read-only.
- **B L1:** A misused bash call can touch anything the user can on the whole machine; the per-call approval prompt is the remaining layer, and the model can request any timeout. — [vibe/core/tools/builtins/bash.py:750](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/builtins/bash.py#L750); [vibe/core/utils/shell.py:46-55](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/utils/shell.py#L46-L55) (verified)
  - *To reach the next level:* Tools are not scoped to the workspace or quantity-bounded; bash has no ceiling on the model-supplied timeout.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

There is no sandbox. Shell commands, hooks, custom Python tools and MCP servers all run directly on the host as the user, with the full environment and full network access. The approval prompt decides whether a command runs, but once it runs nothing contains it, and commands that look read-only (or a test runner the user approves) execute whatever the repository contains.

- **S L0:** Commands run as a same-user host subprocess via create_subprocess_shell. — [vibe/core/utils/shell.py:46-55](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/utils/shell.py#L46-L55); searched `rg -n -i 'sandbox|seatbelt|landlock|bwrap|bubblewrap|seccomp|firejail'` in `vibe` → 2 hits (Both hits are unrelated (a protocol field and a comment in app_server); no isolation backend exists in the Python package.) (verified)
  - *To reach the next level:* No OS-level isolation primitive (container, Seatbelt, Landlock) exists.
- **C L0:** No execution path (bash, hooks, custom tools, MCP stdio servers) is sandboxed. — [vibe/core/hooks/executor.py:44-51](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/hooks/executor.py#L44-L51); [vibe/core/tools/manager.py:205-214](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/manager.py#L205-L214) (verified)
  - *To reach the next level:* No execution path is routed through any isolation boundary.
- **D L0:** No sandbox exists to enable. — searched `rg -n -i 'sandbox|seatbelt|landlock|bwrap|bubblewrap|seccomp|firejail'` in `vibe` → 2 hits (Both hits are unrelated (a protocol field and a comment in app_server); no isolation backend exists in the Python package.) (verified)
  - *To reach the next level:* A sandbox would need to exist and be on by default.
- **B L0:** Executed code has the user's home directory, ~/.ssh and cloud credentials, the full environment and unrestricted network. — [vibe/core/utils/shell.py:58-59](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/utils/shell.py#L58-L59) (verified)
  - *To reach the next level:* Workspace-only mounts, no secrets in the environment and restricted egress would be needed.
- **Cap:** none

### C5 Untrusted input blast radius — 0.33 (high)

Content from web pages, repository files and MCP tool results enters the conversation with the same standing as the user's instructions; nothing tracks where content came from. What limits a hijacked agent is the general approval prompt: shell, web fetch to a new origin, web search and MCP calls all ask the user. File edits inside the project do not, and persistent agent configuration is not protected from unattended edits.

- **S L2:** Exec and egress tools always require approval (regardless of provenance) while in-workspace edits run unattended. — [vibe/core/tools/builtins/bash.py:410-411](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/builtins/bash.py#L410-L411); [vibe/core/tools/builtins/web_fetch.py:82-83](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/builtins/web_fetch.py#L82-L83); [vibe/core/agents/models.py:88-99](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/agents/models.py#L88-L99) (verified)
  - *To reach the next level:* No taint tracking forces approval or disables egress once untrusted content is read; edits stay unattended.
- **C L1:** Untrusted sources are not distinguished; the approval limit applies to actions from any source, but nothing marks tool results or MCP descriptions as data. — searched `rg -n -i 'untrusted|taint|provenance'` in `vibe/core/agent_loop vibe/core/tools` → 1 hits (Single hit is a comment about untrusted cwd config discovery; tool results carry no provenance or taint flag.); [vibe/core/tools/mcp/tools.py:493-500](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/mcp/tools.py#L493-L500) (verified)
  - *To reach the next level:* Tool results, MCP tool descriptions and fetched pages carry no untrusted marking that any control acts on.
- **D L1:** The approval limit is on by default, but an env var disables it and its default-on state is not tamper-resistant. — [vibe/core/config/layers/project.py:91-97](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/config/layers/project.py#L91-L97) (verified)
  - *To reach the next level:* The limit would need to be immune to anything the model can change.
- **B L1:** Exfiltration and shell execution need approval, but a hijacked agent can make unattended edits with persistent consequences; already-approved origins accept further requests unattended. — [vibe/core/agents/models.py:88-99](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/agents/models.py#L88-L99); [vibe/core/tools/builtins/web_fetch.py:125-135](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/builtins/web_fetch.py#L125-L135) (verified)
  - *To reach the next level:* Irreversible or persistent changes would need human approval so that only reversible edits happen unattended.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.25 (high)

Vibe has a real workspace-trust step: project config, hooks, custom tools, skills, agents and AGENTS.md from a repository are ignored until the user trusts the folder, and the trust prompt lists the files it found. After that decision, however, project configuration is not integrity-protected, so poisoned project configuration can persist.

- **S L1:** Trusted-folder config and instruction files load silently, and they are not integrity-protected. — [vibe/core/config/layers/project.py:91-97](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/config/layers/project.py#L91-L97); [vibe/core/config/harness_files/_harness_manager.py:135-142](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/config/harness_files/_harness_manager.py#L135-L142); [vibe/core/tools/manager.py:149-156](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/manager.py#L149-L156) (verified)
  - *To reach the next level:* Security-relevant project config would need validation and a fresh trust decision when it changes.
- **C L2:** The trust gate covers every project-scoped source (config, hooks, tools, skills, agents, AGENTS.md), but it does not cover every path. — [vibe/core/config/harness_files/_harness_manager.py:96-103](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/config/harness_files/_harness_manager.py#L96-L103); [vibe/core/trusted_folders.py:74-87](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/trusted_folders.py#L74-L87) (verified)
  - *To reach the next level:* Project files would need re-review when they change.
- **D L2:** Single-user local CLI: sessions are stored per user under ~/.vibe with owner-only permissions; project config is per directory. — [vibe/core/session/session_logger.py:81](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/session/session_logger.py#L81) (verified)
  - *To reach the next level:* Project-scope settings that affect isolation and security are not protected from change.
- **B L1:** Poisoned project config persists across the user's sessions in that folder and can run hooks or load in-process tools. — [vibe/core/hooks/executor.py:44-51](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/hooks/executor.py#L44-L51); [vibe/core/tools/manager.py:205-214](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/manager.py#L205-L214) (verified)
  - *To reach the next level:* Poisoned config would need to be limited to text influence or purged automatically.
- **Cap:** G2 — The workspace-trust decision is not tamper-resistant at runtime.

### C7 Third-party extensions — 0.07 (high)

Vibe loads three kinds of third-party code: MCP servers launched from config, custom Python tool files, and skills. None is pinned or integrity-checked. Custom tool files from the user's directory or a trusted project's .vibe/tools are imported straight into Vibe's own process at startup, with all its credentials. MCP stdio servers run as separate processes; the MCP SDK gives them a reduced default environment unless config supplies one. A trusted project can add any of these.

- **S L1:** Extensions come from user- or project-chosen commands and files with no version pin or hash. — searched `rg -n -i 'sha256|integrity|signature|checksum'` in `vibe/core/tools/mcp vibe/core/config/mcp_servers.py` → 2 hits (Both hits hash config/descriptor cache keys; no extension is pinned or integrity-checked.); [vibe/core/tools/mcp/tools.py:393-396](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/mcp/tools.py#L393-L396) (verified)
  - *To reach the next level:* No version pinning or integrity check exists for MCP servers or tool files.
- **C L0:** No extension type is verified. — searched `rg -n -i 'sha256|integrity|signature|checksum'` in `vibe/core/tools/mcp vibe/core/config/mcp_servers.py` → 2 hits (Both hits hash config/descriptor cache keys; no extension is pinned or integrity-checked.) (verified)
  - *To reach the next level:* At least one extension type would need verification.
- **D L0:** Nothing third-party ships enabled, but a trusted folder's .vibe/tools, hooks and mcp_servers config are loaded with no prompt showing what will run, and that configuration is not integrity-protected. — [vibe/core/tools/manager.py:149-156](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/manager.py#L149-L156); [vibe/core/agents/models.py:88-99](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/agents/models.py#L88-L99); [vibe/core/tools/manager.py:471](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/manager.py#L471) (verified)
  - *To reach the next level:* Adding an extension would need explicit consent showing the exact command, and the workspace should not be able to add extensions.
- **B L0:** Custom Python tools are exec_module'd into the agent process, inheriting its API key and full environment. — [vibe/core/tools/manager.py:205-214](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/manager.py#L205-L214) (verified)
  - *To reach the next level:* Extensions would need to run out of process with a scrubbed environment.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.40 (high)

The Mistral API key is stored in the OS keyring by default (falling back to ~/.vibe/.env), crash reports drop local variables and scrub paths, telemetry events carry metadata rather than prompt or tool content, and session transcripts are written owner-only. Reading .env files needs approval. But nothing redacts secrets from tool output before it goes to the model or into the transcript, the shell inherits the full environment (including any exported keys), and telemetry is on by default.

- **S L2:** Keyring storage, .env sensitive-file prompts and Sentry scrubbing exist, but no redaction is applied to model-bound messages or transcripts. — [vibe/setup/auth/api_key_persistence.py:204-210](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/setup/auth/api_key_persistence.py#L204-L210); [vibe/core/tools/utils.py:36-43](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/utils.py#L36-L43); [vibe/observability/sentry.py:190-197](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/observability/sentry.py#L190-L197); searched `rg -n -i 'redact|mask|secret'` in `vibe/core/session/session_logger.py vibe/core/agent_loop/_loop.py` → 1 hits (Only hit is a comment about owner-only file mode; no redaction of transcripts or model-bound tool output.) (verified)
  - *To reach the next level:* Redaction before logs and model-bound messages on all major paths is missing.
- **C L2:** Crash reports, telemetry and transcript file permissions are protected; model-bound messages and subprocess environments are not. — [vibe/core/utils/shell.py:58-59](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/utils/shell.py#L58-L59); [vibe/core/session/session_logger.py:387-395](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/session/session_logger.py#L387-L395) (verified)
  - *To reach the next level:* Model-bound messages and subprocess environments are unprotected.
- **D L1:** enable_telemetry defaults to true; the tool-call event sends tool name, decision and file counts, not content. — [vibe/core/config/vibe_schema.py:592](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/config/vibe_schema.py#L592); [vibe/core/telemetry/send.py:374-386](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/telemetry/send.py#L374-L386) (verified)
  - *To reach the next level:* Telemetry would need to be opt-in.
- **B L1:** A leaked Mistral key is long-lived and account-scoped; any user credentials exported in the environment reach every subprocess. — [vibe/core/utils/shell.py:58-59](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/utils/shell.py#L58-L59) (verified)
  - *To reach the next level:* Keys reachable from the agent are not scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.50 (high)

Every session is recorded as a JSON-lines transcript of all messages, including each tool call and its result, under ~/.vibe/logs/session with owner-only permissions, flushed and fsynced after every model step. Sub-agent sessions are linked to the parent tool call. The record is outside the project folder, but the agent's shell could still edit it (with an outside-workdir approval), it is not tamper-evident, and approval decisions go to the log and telemetry rather than as structured fields in the transcript.

- **S L2:** A structured messages.jsonl transcript records every tool call and result. — [vibe/core/session/session_logger.py:384-401](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/session/session_logger.py#L384-L401); [vibe/core/agent_loop/_loop.py:2101-2105](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/agent_loop/_loop.py#L2101-L2105) (verified)
  - *To reach the next level:* No actor/approver attribution in the transcript and no tamper-evidence.
- **C L2:** All tool calls, including MCP and sub-agent sessions (linked by tool_call_id), are in the transcript. — [vibe/core/agent_loop/_loop.py:980-994](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/agent_loop/_loop.py#L980-L994) (verified)
  - *To reach the next level:* Approvals and denials are not recorded as structured transcript entries.
- **D L2:** On by default and stored under ~/.vibe/logs/session, outside the workspace, but in a path the agent's own shell can write. — [vibe/core/paths/_vibe_home.py:13](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/paths/_vibe_home.py#L13); [vibe/core/config/models.py:71-73](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/config/models.py#L71-L73) (verified)
  - *To reach the next level:* The record is written by the same process and user the model drives.
- **B L2:** Writes are fsynced per step and failures raise errors. — [vibe/core/session/session_logger.py:396-405](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/session/session_logger.py#L396-L405) (verified)
  - *To reach the next level:* Records are flushed per model step, not per action, and actions do not fail closed on a logging error.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

Turn, cost and token limits exist but only apply in programmatic (-p) mode; the default interactive session has no step, time or spend ceiling. Shell commands default to a 5-minute timeout, but the model can ask for any longer timeout. Stopping works well: Escape cancels the turn and running shell commands are killed with their whole process group. Sub-agents cannot spawn further sub-agents.

- **S L2:** Turn, price and session-token middleware plus per-command timeouts are enforced in code, and cancellation kills the process group. — [vibe/core/agent_loop/_loop.py:1829-1840](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/agent_loop/_loop.py#L1829-L1840); [vibe/core/utils/async_subprocess.py:53](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/utils/async_subprocess.py#L53); searched `rg -n -i 'wall.?clock|session_timeout|max_duration|rate.?limit'` in `vibe/core/agent_loop vibe/core/tools` → 6 hits (All hits are handling of provider RateLimitError; no session wall-clock limit or side-effect rate limiter.) (verified)
  - *To reach the next level:* No session wall-clock limit and no rate limit on side-effecting tools.
- **C L2:** Limits apply to the loop and shell timeouts; sub-agent depth is capped at 1 and children receive the parent's runtime policy. — [vibe/core/tools/builtins/task.py:97-101](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/builtins/task.py#L97-L101); [vibe/core/agent_loop/_loop.py:962-978](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/agent_loop/_loop.py#L962-L978) (verified)
  - *To reach the next level:* Sub-agents do not draw from one shared budget with the parent.
- **D L1:** --max-turns/--max-price/--max-tokens apply only with -p; interactive sessions are unlimited and the model sets its own bash timeout. — [vibe/cli/entrypoint.py:74-95](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/cli/entrypoint.py#L74-L95); [vibe/core/tools/builtins/bash.py:750](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/builtins/bash.py#L750) (verified)
  - *To reach the next level:* Sensible step/cost defaults in interactive mode and a ceiling on model-chosen timeouts are missing.
- **B L1:** A runaway interactive session has no time or spend ceiling; stopping does kill in-flight shell processes. — [vibe/core/tools/builtins/bash.py:816-818](https://github.com/mistralai/mistral-vibe/blob/7c19608af06f6c61d63f8f7a5c3430da73fba2ab/vibe/core/tools/builtins/bash.py#L816-L818) (verified)
  - *To reach the next level:* No per-run time or cost ceiling in the default mode.
- **Cap:** G1 — Turn, cost and token limits are opt-in and only take effect in programmatic mode, not in the default interactive session.

## Rule-of-Two check
[A] untrusted input: web_fetch/web_search results, repository files and MCP outputs enter context unmarked (vibe/core/tools/builtins/web_fetch.py:138) · [B] sensitive data/systems: user filesystem and full process environment reachable from bash (vibe/core/utils/shell.py:59) · [C] state change / egress: unattended file edits (vibe/core/agents/models.py:96) and approved bash/web_fetch (vibe/core/tools/builtins/bash.py:411) · Same default session? Yes

## Highest-impact improvements
1. Harden the default accept-edits profile and accept approval-disabling settings only from user scope. — C2 D L1→L3, +0.100 before caps (Playbook 5)
2. Strengthen the workspace-trust model so security-relevant project configuration is re-reviewed when it changes. — C6 S L1→L2, +0.075 before caps (Playbook 2)
3. Apply default turn/cost limits to interactive sessions and cap the model-supplied bash timeout. — C10 D L1→L2, +0.050 before caps (Playbook 3 step 3)
4. Strip provider API keys and other known secret variables from the environment passed to shell subprocesses. — C8 C L2→L3, +0.075 before caps (Playbook 4)
5. Offer an OS sandbox (Seatbelt/Landlock/bubblewrap) for bash with workspace-only writes and no network by default. — C4 S L0→L3, +0.225 before caps (Playbook 3 step 1)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the default Python TUI on the legacy backend. The Rust CLI (VIBE_CLI=rust), the Unified Harness (harness/, selected by --experimental-harness or a GrowthBook rollout when the internal module is installed), its smart-approve LLM classifier and plugin system, and the ACP / app-server entry points were not scored.
- A remote GrowthBook experiment can switch eligible users to the Unified Harness and to smart-approve as default agent; that path was not reviewed.
- MCP stdio servers' reduced default environment relies on the mcp Python SDK's behaviour when env is None; not verified in the SDK source.
- The timing of one finding (same-session versus next-session effect) was not established.
- No reviewer-injection text was found in the repository.
