# Defense-in-Depth Score: Serena

**Repo:** https://github.com/oraios/serena · **Commit:** `d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809` · **Reviewed:** 2026-10-03
**What it is:** MCP coding toolkit: LSP-based semantic retrieval and editing
**Category:** Coding
**Scored configuration:** `serena start-mcp-server` with no flags: stdio transport, default desktop-app context (all non-optional tools including execute_shell_command), LSP backend, fresh serena_config.yml from the shipped template.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents no · external communication yes

## Score: 2.3 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C3 | Tool & action scoping | L2 | L1 | L1 | L0 | 0.28 | — | **0.28** | High |
| C4 | Code-execution isolation | L2 | L3 | L0 | L2 | 0.47 | G1 | **0.47** (alt) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L1 | L1 | L0 | 0.12 | C6-REPOCONFIG | **0.12** | High |
| C7 | Third-party extensions | L2 | L1 | L0 | L1 | 0.28 | — | **0.28** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |


As shipped, Serena gives the connected model an unrestricted shell that runs as your user with your full environment. It adds file-editing tools whose project-folder boundary the model can move with activate_project. The server provides no sandbox, approval step, or provenance marking, so safety rests on the MCP client's approval prompts and on trusting the repository. A repository's committed .serena/project.yml and memories can inject instructions and enable tools without a trust decision. The maintainers say this openly: their security docs assume the repository and LLM are trusted and recommend a container for real isolation.

## Critical gaps
- The default-on shell tool runs any model-supplied command on the host as the OS user with the full inherited environment. (ASI03, ASI02; C1) — [src/serena/util/shell.py:30-43](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/util/shell.py#L30-L43); [src/serena/tools/cmd_tools.py:27](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/cmd_tools.py#L27)
- No execution isolation by default: shell commands are same-user host subprocesses; the Docker setup is opt-in and experimental. (ASI05; C4) — [src/serena/util/shell.py:30-43](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/util/shell.py#L30-L43); [src/serena/resources/config/contexts/desktop-app.yml:11](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/resources/config/contexts/desktop-app.yml#L11)
- A hijacked session can both exfiltrate (via shell network tools) and take irreversible actions with no server-side gate. (ASI01; C5) — [src/serena/util/shell.py:30-43](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/util/shell.py#L30-L43); [src/serena/agent.py:1253-1254](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/agent.py#L1253-L1254)
- Repository-controlled .serena/project.yml can enable tools and select language servers to launch without a trust decision. (ASI06, ASI04; C6) — [src/serena/agent.py:1299](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/agent.py#L1299); [src/serena/agent.py:183](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/agent.py#L183); [src/serena/resources/project.template.yml:41](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/resources/project.template.yml#L41)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

Serena runs as the OS user who launched it and does nothing to narrow that authority. Its shell tool, which is on by default, runs any command with the user's full environment, and language-server processes get a copy of the whole environment too. There is no per-request authorization layer and no read-only default; the only guidance against misuse is a line in the tool description asking the model not to run unsafe commands. If the model is steered by malicious content, it can use everything the user's account can reach, including cloud CLIs and keys in the environment.

- **S L0:** Ambient OS-user authority; the shell subprocess inherits the full parent environment (no env= argument) and language servers get os.environ.copy(). — [src/serena/util/shell.py:30-43](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/util/shell.py#L30-L43); [src/solidlsp/util/subprocess_util.py:212](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/solidlsp/util/subprocess_util.py#L212) (verified)
  - *To reach the next level:* L1 needs a dedicated identity for the server's actions instead of the user's ambient authority.
- **C L0:** The only restraint on the shell tool is a prompt sentence in its description; no authorization check runs before any tool. — [src/serena/tools/cmd_tools.py:41](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/cmd_tools.py#L41); searched `rg -n -i 'authoriz|permission'` in `src/serena/tools` → 1 hits (the single hit is a docstring asking the model to delete memories only with user permission; no authorization logic in any tool) (verified)
  - *To reach the next level:* L1 needs a code-level authorization check on at least the main tool path.
- **D L0:** The default desktop-app context excludes nothing, so the shell and all edit tools run with the user's privilege on a fresh install. — [src/serena/constants.py:26](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/constants.py#L26); [src/serena/resources/config/contexts/desktop-app.yml:11](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/resources/config/contexts/desktop-app.yml#L11) (verified)
  - *To reach the next level:* L1 needs a narrower default (e.g. no shell) that operators widen deliberately.
- **B L0:** A hijacked session reaches everything the OS user can: local files, SSH keys, cloud CLIs and any tokens in the environment. — [src/serena/util/shell.py:30-43](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/util/shell.py#L30-L43); [src/serena/repl/api/shell_api.py:78-79](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/repl/api/shell_api.py#L78-L79) (verified)
  - *To reach the next level:* L1 would require the reachable authority to be limited below the user's whole account.
- **Cap:** none

### C2 Approval gates — 0.20 (high)

As an MCP server, Serena leaves approval to the client and gives it only MCP hints. Tools inherit a read-only or destructive hint from an internal 'can edit' marker. The shell and file-edit tools are correctly flagged, but not every tool's hint is accurate. There is no dry-run or diff preview, and no confirmation step the server enforces. A read-only project mode exists but is off by default and set in the project's own repository file. Wrongly approved shell commands can make irreversible changes, and Serena keeps no checkpoint to undo them.

- **S L1:** MCP readOnly/destructive hints are derived from the CanEdit marker (replace_in_files also offers a model-chosen dry_run), but not every tool's hint is accurate. — [src/serena/mcp.py:109-113](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/mcp.py#L109-L113) (verified)
  - *To reach the next level:* L2 needs accurate hints on every mutating tool.
- **C L1:** The marker covers shell and edit tools, but hint coverage does not extend to every other mutating tool. — [src/serena/tools/cmd_tools.py:27](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/cmd_tools.py#L27) (verified)
  - *To reach the next level:* L2 needs every mutating path flagged for the host's gate.
- **D L1:** Hints are on by default, but the tool set the host sees can be widened by a repository-controlled .serena/project.yml (included_optional_tools is applied without a trust check). — [src/serena/agent.py:1299](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/agent.py#L1299); [src/serena/agent.py:851](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/agent.py#L851); [src/serena/agent.py:183](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/agent.py#L183) (verified)
  - *To reach the next level:* L2 needs the risk signalling and tool set to be unaffected by repository files.
- **B L0:** A wrongly approved shell call can delete data, force-push or send data out; the server keeps no checkpoint or undo. — [src/serena/util/shell.py:30-43](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/util/shell.py#L30-L43); searched `rg -n -i 'checkpoint|undo|rollback|snapshot'` in `src/serena` → 10 hits (hits are LSP diagnostics snapshots and a config-merge comment; no checkpoint or undo for edits or shell); [src/serena/tools/file_tools.py:176](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/file_tools.py#L176) (verified)
  - *To reach the next level:* L1 needs at least some actions to be reversible (checkpoints for file edits).
- **Cap:** none

### C3 Tool & action scoping — 0.28 (high)

Serena's file and symbol tools check that paths stay inside the active project. The check is lexical, and the code allows symlinks on purpose. Two default-on tools make this boundary weak. The shell tool accepts any command string and any absolute working directory. activate_project lets the model make any directory, such as the home directory, the new project root. The default context exposes all non-optional tools, including shell and editing. Users can exclude tools globally or pick a narrower context.

- **S L2:** Path containment is normpath+commonpath without resolving symlinks (symlinks intentionally allowed); the shell tool takes a raw command string. — [src/serena/project.py:302-311](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/project.py#L302-L311); [src/serena/util/shell.py:30-43](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/util/shell.py#L30-L43) (verified)
  - *To reach the next level:* L3 needs resolved-path containment and replacing the raw shell with narrow tools or allowlisted commands.
- **C L1:** File and symbol tools validate paths, but the default-on shell (any command, any absolute cwd) and activate_project (any directory becomes the root) sidestep the check. — [src/serena/repl/api/fs_api.py:193](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/repl/api/fs_api.py#L193); [src/serena/repl/api/shell_api.py:78-79](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/repl/api/shell_api.py#L78-L79); [src/serena/agent.py:1530-1531](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/agent.py#L1530-L1531) (verified)
  - *To reach the next level:* L2 needs the most powerful tools to pass the same validation instead of bypassing it.
- **D L1:** Shell, file write and edit tools are all on in the default desktop-app context; tools can be excluded individually or via other contexts, and repo project.yml can add tools. — [src/serena/resources/config/contexts/desktop-app.yml:11](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/resources/config/contexts/desktop-app.yml#L11); [src/serena/tools/cmd_tools.py:27](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/cmd_tools.py#L27); [src/serena/agent.py:1299](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/agent.py#L1299) (verified)
  - *To reach the next level:* L2 needs a default tool group without exec.
- **B L0:** The shell tool reaches the whole machine as the user. — [src/serena/util/shell.py:30-43](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/util/shell.py#L30-L43); [src/serena/repl/api/shell_api.py:78-79](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/repl/api/shell_api.py#L78-L79) (verified)
  - *To reach the next level:* L1 needs the general tools bounded below whole-machine reach.
- **Cap:** none

### C4 Code-execution isolation — 0.47 (high)

The default shell tool runs model-written commands on the host as the same user, with no sandbox and the full environment. The optional REPL tool runs Python inside the Serena process. Serena ships an experimental Docker image that its security docs recommend as the only real protection. It is opt-in, runs as root with default capabilities and full network, and mounts the workspace read-write. That still beats the default, so the criterion takes the opt-in score, capped because it is off by default.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Same-user host subprocess via Popen(shell=True); no isolation primitive anywhere in the server. — [src/serena/util/shell.py:30-43](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/util/shell.py#L30-L43); searched `rg -n -i 'seccomp|landlock|bwrap|firejail|chroot|nsjail|gvisor'` in `src/serena src/solidlsp` → 0 hits (no sandbox primitive) (verified)
    - *To reach the next level:* L1 needs at least a filtering layer or separate working directory; L2 needs OS-level separation.
  - **C L0:** No execution path (shell, REPL, language servers) is isolated. — [src/serena/util/shell.py:30-43](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/util/shell.py#L30-L43); [src/solidlsp/util/subprocess_util.py:212](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/solidlsp/util/subprocess_util.py#L212) (verified)
    - *To reach the next level:* L1 needs the main exec tool sandboxed.
  - **D L0:** There is no default isolation; the shell tool is enabled in the default context. — [src/serena/resources/config/contexts/desktop-app.yml:11](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/resources/config/contexts/desktop-app.yml#L11); [src/serena/tools/cmd_tools.py:27](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/cmd_tools.py#L27) (verified)
    - *To reach the next level:* L1 needs isolation on by default.
  - **B L0:** Commands run host-equivalent: user home, credentials and environment, full network. — [src/serena/util/shell.py:30-43](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/util/shell.py#L30-L43); [src/solidlsp/util/subprocess_util.py:212](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/solidlsp/util/subprocess_util.py#L212) (verified)
    - *To reach the next level:* L1 needs at least the home directory and credentials kept out of reach.
- **opt-in experimental Docker container (compose.yaml)** (alt; raw 0.47, cap G1 → 0.47) ← counted
  - **S L2:** Stock python:3.11-slim container with no USER directive, so processes run as root with default capabilities. — [Dockerfile:2](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/Dockerfile#L2); searched `rg -n '^USER'` in `Dockerfile` → 0 hits (no non-root user) (verified)
    - *To reach the next level:* L3 needs a hardened container (non-root, dropped capabilities, no-new-privileges, read-only root).
  - **C L3:** The whole server, including the shell tool and language servers, runs inside the container, so every model-reachable path is contained. — [compose.yaml:18](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/compose.yaml#L18); [compose.yaml:16](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/compose.yaml#L16) (verified)
    - *To reach the next level:* L4 needs fail-closed behaviour; nothing stops the user running Serena on the host instead.
  - **D L0:** Opt-in and documented as experimental; the default launch runs on the host. — [compose.yaml:18](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/compose.yaml#L18); [src/serena/cli.py:241](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/cli.py#L241) (verified)
    - *To reach the next level:* L1 needs the container to be the default.
  - **B L2:** Workspace mounted read-write with unrestricted network; host credentials are not mounted by default. — [compose.yaml:4-6](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/compose.yaml#L4-L6); [compose.yaml:13](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/compose.yaml#L13) (verified)
    - *To reach the next level:* L3 needs egress off or allowlisted and resource limits.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.00 (high)

Serena reads untrusted content all the time: source files, search results, shell output, committed memories and the repository's .serena/project.yml. None of it is marked as untrusted. The project's initial_prompt is placed into the activation result as project instructions, and the tool descriptions tell the model to read memories before running commands. A repository author can therefore write directly to the model. The server offers no structural limit, so a hijacked session can run shell commands to leak data and make irreversible changes, unless the MCP client happens to gate them.

- **S L0:** Tool outputs mix repository content with directives: the repo's initial_prompt is rendered into activate_project output as project-instructions, and memory lists come with read-these instructions. — [src/serena/agent.py:1253-1254](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/agent.py#L1253-L1254); [src/serena/agent.py:1233-1240](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/agent.py#L1233-L1240); [src/serena/tools/cmd_tools.py:40](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/cmd_tools.py#L40) (verified)
  - *To reach the next level:* L1 needs plain outputs without injected directives from repository content.
- **C L0:** No source is distinguished; file contents, memories and shell output return as plain tool results. — searched `rg -n -i 'untrusted|provenance' -g '*.py'` in `src/serena` → 0 hits (no untrusted-content marking) (verified)
  - *To reach the next level:* L1 needs at least one untrusted source handled distinctly.
- **D L0:** No control exists to be on by default; the read-only project mode is opt-in. — [src/serena/resources/project.template.yml:123](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/resources/project.template.yml#L123) (verified)
  - *To reach the next level:* L1 needs a default-on limit.
- **B L0:** A hijacked model can exfiltrate via shell (curl, git push) and run irreversible commands with no server-side human step. — [src/serena/util/shell.py:30-43](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/util/shell.py#L30-L43); [src/serena/tools/cmd_tools.py:27](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/cmd_tools.py#L27) (verified)
  - *To reach the next level:* L1 needs either exfiltration or irreversible action to require approval.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.12 (high)

Serena loads state the repository controls. .serena/project.yml can inject an initial prompt, add tools and choose which language servers are downloaded and started. Memories under .serena/memories are designed to be committed and shared, and are offered to the model on activation. A trust setting, empty by default on new installs, gates only two project settings: the activation command and language-server overrides. The model can write and edit project and global memories freely; read-only patterns exist but are empty by default. A poisoned memory can therefore persist across sessions and spread to everyone who clones the repository.

- **S L0:** Repo-controlled project.yml adds tools (included_optional_tools) and launches language servers (language_servers) without a trust decision; model-written memories are re-offered as context. — [src/serena/agent.py:1299](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/agent.py#L1299); [src/serena/agent.py:183](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/agent.py#L183); [src/serena/resources/project.template.yml:41](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/resources/project.template.yml#L41); [src/serena/agent.py:1233-1240](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/agent.py#L1233-L1240) (verified)
  - *To reach the next level:* L1 needs repo config unable to add tools/servers silently and memory writes at least logged distinctly.
- **C L1:** Trust gating covers activation_command and ls_specific_settings only; tools, prompts, language servers and memories are ungated. — [src/serena/agent.py:1483](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/agent.py#L1483); [src/serena/project.py:524](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/project.py#L524); [src/serena/resources/serena_config.template.yml:241](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/resources/serena_config.template.yml#L241) (verified)
  - *To reach the next level:* L2 needs the main memory store and tool-enabling settings controlled.
- **D L1:** Memories are namespaced per project directory plus a writable global namespace shared across all projects; no read-only patterns by default. — [src/serena/memories/memory_manager.py:48](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/memories/memory_manager.py#L48); [src/serena/config/serena_config.py:116](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/config/serena_config.py#L116); [src/serena/resources/serena_config.template.yml:142](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/resources/serena_config.template.yml#L142); [src/serena/memories/memory_manager.py:202-205](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/memories/memory_manager.py#L202-L205) (verified)
  - *To reach the next level:* L2 needs namespaces the model cannot cross (global namespace read-only by default).
- **B L0:** Project memories live in the repo's .serena folder, which is not gitignored, so poisoned memories propagate to other users and can steer shell use. — [src/serena/resources/serena_config.template.yml:222](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/resources/serena_config.template.yml#L222); [src/serena/project.py:72-73](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/project.py#L72-L73); [src/serena/tools/cmd_tools.py:40](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/cmd_tools.py#L40) (verified)
  - *To reach the next level:* L1 needs persistence limited to one user.
- **Cap:** C6-REPOCONFIG — A repository's .serena/project.yml can, without any trust decision, enable tools (included_optional_tools is applied to the active tool set) and choose which language-server processes are downloaded and launched.

### C7 Third-party extensions — 0.28 (high)

With the default LSP backend, Serena downloads and runs third-party language servers on its own when a project is activated. Which servers run depends on languages it detects in the repository or on the repository's project.yml. Versions are pinned, and archive downloads are restricted by host and checked against a SHA-256 hash when one is configured. npm-based servers are pinned but installed without a digest. The servers run as separate processes under the same user with the full environment. The user is not asked before installation.

- **S L2:** Language-server versions are pinned; archive downloads verify SHA-256 only when a hash is configured, and npm installs are pinned without a digest. — [src/solidlsp/ls_utils.py:621-623](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/solidlsp/ls_utils.py#L621-L623); [src/solidlsp/language_servers/common.py:156-162](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/solidlsp/language_servers/common.py#L156-L162) (verified)
  - *To reach the next level:* L3 needs integrity verification on every download path.
- **C L1:** Only archive-based downloads go through the host-allowlisted, hash-checking path (and only when a hash is configured); npm/pip-installed servers rely on version pins only. — [src/solidlsp/language_servers/common.py:143](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/solidlsp/language_servers/common.py#L143); [src/solidlsp/language_servers/common.py:156-162](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/solidlsp/language_servers/common.py#L156-L162) (verified)
  - *To reach the next level:* L2 needs integrity checks on most install paths, including npm-based servers.
- **D L0:** Language servers are installed automatically on activation, selected from detected languages or the repository's project.yml, with no consent prompt. — [src/serena/resources/project.template.yml:41](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/resources/project.template.yml#L41); [src/serena/agent.py:1299](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/agent.py#L1299); [src/solidlsp/language_servers/typescript_language_server.py:299](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/solidlsp/language_servers/typescript_language_server.py#L299) (verified)
  - *To reach the next level:* L1 needs at least a consent prompt before first install.
- **B L1:** Servers run as separate processes with the user's full environment copied in. — [src/solidlsp/util/subprocess_util.py:212](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/solidlsp/util/subprocess_util.py#L212) (verified)
  - *To reach the next level:* L2 needs a scrubbed environment for launched servers.
- **Cap:** none
- **Notes:** G1 does not apply: version pinning and host/hash checks are always on; D is L0 because installation happens automatically without consent.

### C8 Secrets & sensitive-data protection — 0.15 (high)

Serena holds almost no secrets of its own. Its local authentication secret is hidden from repr and kept in a config file restricted to its owner. Everything else is unprotected. Full tool arguments and results, including any secret a file read returns, are logged at INFO to ~/.serena/logs and to the dashboard. read_file can open gitignored files such as .env. The shell and language servers inherit the full environment. A content-free usage ping is sent by default unless SERENA_USAGE_REPORTING=false.

- **S L1:** The one secret field is repr-hidden and the config file is chmod 0600; there is no log or output redaction. — [src/serena/config/serena_config.py:890](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/config/serena_config.py#L890); [src/serena/config/serena_config.py:1066-1071](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/config/serena_config.py#L1066-L1071); searched `rg -n -i 'redact'` in `src/serena` → 0 hits (no redaction) (verified)
  - *To reach the next level:* L2 needs log filters on main paths.
- **C L1:** Only the config representation path is protected; logs, model-bound tool results and subprocess environments are not. — [src/serena/tools/tools_base.py:377](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/tools_base.py#L377); [src/solidlsp/util/subprocess_util.py:212](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/solidlsp/util/subprocess_util.py#L212) (verified)
  - *To reach the next level:* L2 needs logs and transcripts protected.
- **D L0:** Full tool payload logging is on by default (INFO level file log), and a usage ping is on by default. — [src/serena/tools/tools_base.py:256](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/tools_base.py#L256); [src/serena/tools/tools_base.py:377](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/tools_base.py#L377); [src/serena/cli.py:361-364](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/cli.py#L361-L364); [src/serena/agent.py:751-762](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/agent.py#L751-L762) (verified)
  - *To reach the next level:* L1 needs payload logging off or redacted by default.
- **B L0:** Any long-lived keys in the user's environment are reachable by every shell subprocess the model starts. — [src/serena/util/shell.py:30-43](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/util/shell.py#L30-L43); [src/solidlsp/util/subprocess_util.py:212](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/solidlsp/util/subprocess_util.py#L212) (verified)
  - *To reach the next level:* L1 needs subprocess environments scrubbed so only scoped keys are reachable.
- **Cap:** none
- **Notes:** G1 does not apply: the repr masking and 0600 config are always on; D is L0 because full tool-payload logging is on by default.

### C9 Audit & traceability — 0.50 (high)

Every tool call goes through one wrapper. It logs the tool name and all arguments before running and logs the result afterwards, with timestamps, to a per-run file under ~/.serena/logs. The file is written by default, outside the project folder. It is plain text with no actor attribution, nothing makes it tamper-evident, and the shell tool can edit or delete it.

- **S L2:** Each call logs tool name, full parameters and result at INFO with timestamps via SERENA_LOG_FORMAT. — [src/serena/tools/tools_base.py:256](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/tools_base.py#L256); [src/serena/tools/tools_base.py:377](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/tools_base.py#L377) (verified)
  - *To reach the next level:* L3 needs actor attribution (requesting client/principal) and correlation IDs.
- **C L2:** All tools go through Tool.apply_ex, which logs before execution. — [src/serena/tools/tools_base.py:332-333](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/tools_base.py#L332-L333) (verified)
  - *To reach the next level:* L3 needs approvals/denials and config changes recorded.
- **D L2:** The file handler is added unconditionally at start-mcp-server, writing to ~/.serena/logs, outside the workspace but writable by the shell tool. — [src/serena/cli.py:361-364](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/cli.py#L361-L364); [src/serena/config/serena_config.py:133](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/config/serena_config.py#L133) (verified)
  - *To reach the next level:* L3 needs the record written where the model's tools can't alter it.
- **B L2:** Python logging flushes per record and reports handler errors to stderr; actions proceed if logging fails. — [src/serena/cli.py:361-364](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/cli.py#L361-L364) (verified)
  - *To reach the next level:* L3 needs a replayable durable trajectory.
- **Cap:** none

### C10 Limits & kill switch — 0.33 (high)

Serena applies a 240-second timeout and a 150,000-character output cap to every tool call. The timeout only stops waiting: the worker thread and any shell process keep running, and the shell call itself has no timeout. The model can raise the output cap on each call through max_answer_chars. There are no rate limits.

- **S L2:** Server-enforced tool timeout and output-size cap on all tools. — [src/serena/resources/serena_config.template.yml:154](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/resources/serena_config.template.yml#L154); [src/serena/tools/tools_base.py:391-394](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/tools_base.py#L391-L394); [src/serena/resources/serena_config.template.yml:193](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/resources/serena_config.template.yml#L193) (verified)
  - *To reach the next level:* L3 needs caps on every operation plus rate or concurrency limits.
- **C L1:** The timeout abandons the wait but the thread and spawned shell processes continue. — [src/serena/task_executor.py:92](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/task_executor.py#L92); [src/serena/util/shell.py:43](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/util/shell.py#L43) (verified)
  - *To reach the next level:* L2 needs timeouts that actually stop the tool's work.
- **D L1:** Sensible defaults, but the model can raise the output cap per call. — [src/serena/tools/tools_base.py:263](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/tools/tools_base.py#L263) (verified)
  - *To reach the next level:* L2 needs limits the model cannot raise.
- **B L1:** After a timeout, the shell command keeps running with no process kill. — [src/serena/task_executor.py:92](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/task_executor.py#L92); [src/serena/util/shell.py:30-43](https://github.com/oraios/serena/blob/d0f7f92631c23dc4c5ed0b5ccd35bc623b19a809/src/serena/util/shell.py#L30-L43) (verified)
  - *To reach the next level:* L2 needs in-flight tool work stopped on timeout.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: repository files, committed memories and project.yml initial_prompt (agent.py:1253, fs_api.py:193) · [B] sensitive data/systems: anything the OS user can read, plus the inherited environment (shell.py:30, subprocess_util.py:212) · [C] state change / egress: execute_shell_command and edit tools, default-on (cmd_tools.py:27, shell.py:32) · Same default session? Yes

## Highest-impact improvements
1. Drop execute_shell_command from the default desktop-app context (as the claude-code and ide contexts already do) so shell access is an explicit opt-in. — C3 D L1→L2, +0.050 before caps (Playbook 3)
2. Gate all tool-enabling and prompt-injecting project.yml fields (included_optional_tools, fixed_tools, initial_prompt, language_servers, added_modes) on project trust, like activation_command. — C6 S L0→L1, +0.075 before caps (Playbook 2)
3. Ensure every mutating tool is marked as editing so MCP hosts see it as non-read-only. — C2 S L1→L2, +0.075 before caps (Playbook 5)
4. Pass a timeout to the shell subprocess and kill its process group when the tool timeout expires. — C10 C L1→L2, +0.075 before caps (Playbook 3 step 3)
5. Pass a scrubbed environment to shell and language-server subprocesses. — C7 B L1→L2, +0.050 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the no-flag default (desktop-app context). The README-recommended client contexts (claude-code, ide, codex, etc.) exclude execute_shell_command and some file tools, which narrows C3/C4 exposure; in those single-project contexts a repo's project.yml can re-add excluded tools via included_optional_tools.
- Installations whose serena_config.yml predates the project-trust setting may differ from the fresh-install template value scored here.
- The JetBrains backend, optional REPL interface, HTTP/SSE transports, Claude Code hook helpers (hooks.py) and the per-language-server download code for all ~70 servers were sampled rather than exhaustively reviewed.
- No reviewer-injection attempts found; AGENTS.md and committed .serena memories contain ordinary development instructions.
