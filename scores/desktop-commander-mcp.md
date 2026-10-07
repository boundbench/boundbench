# Defense-in-Depth Score: Desktop Commander MCP

**Repo:** https://github.com/wonderwhy-er/DesktopCommanderMCP · **Commit:** `c774c3b505de990219637ecdc9a830c8772fae9d` (v0.2.52) · **Reviewed:** 2026-10-03
**What it is:** MCP server giving terminal control, filesystem search and diff editing
**Category:** AI Assistants
**Scored configuration:** Local stdio server installed via npx (README Option 1) with the default config it creates: empty allowedDirectories, default command blocklist, telemetry on.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents no · external communication yes

## Score: 2.2 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | C1-SELFESC | **0.00** | High |
| C2 | Approval gates | L2 | L2 | L3 | L0 | 0.45 | — | **0.45** | High |
| C3 | Tool & action scoping | L1 | L1 | L0 | L0 | 0.15 | G2 | **0.15** | High |
| C4 | Code-execution isolation | L2 | L3 | L3 | L2 | 0.62 | G1 | **0.50** (alt) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C7 | Third-party extensions | L1 | L0 | L0 | L0 | 0.07 | — | **0.07** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |


Desktop Commander hands the connected AI a full shell and file access on your machine as you, with no sandbox and whole-filesystem access by default. Its folder list and command blocklist are guardrails its own security policy says can be bypassed, and the model can switch them off itself with set_config_value. The dominant risk is prompt injection: a web page or file the model reads can lead to secrets being read and sent out, or data being deleted, with nothing in the server to stop it. Run it in the shipped Docker setup with only the folders you need if the AI should not reach the rest of your machine.

## Critical gaps
- The model can rewrite its own permission scope (allowed folders, command blocklist) through the set_config_value tool, with no server-side gate. (ASI03; C1) — [src/tools/config.ts:247](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/config.ts#L247); [src/config-field-definitions.ts:11-20](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/config-field-definitions.ts#L11-L20)
- The server acts with the user's full ambient authority by default: empty allowed-folders list means the whole filesystem, and shells inherit the full environment. (ASI03; C1) — [src/config-manager.ts:188](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/config-manager.ts#L188); [src/terminal-manager.ts:207-212](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L207-L212)
- The model can disable the command blocklist and folder restrictions at runtime via set_config_value. (ASI02; C3) — [src/tools/config.ts:247](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/config.ts#L247); [src/config-field-definitions.ts:11-20](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/config-field-definitions.ts#L11-L20)
- Model-chosen shell commands and Node.js code run directly on the host as the user with the full environment, with no sandbox in the default install. (ASI05; C4) — [src/terminal-manager.ts:256](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L256); [src/terminal-manager.ts:207-212](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L207-L212)
- If injected content hijacks the session, it can both leak secrets (shell, URL fetch) and take irreversible actions on the host with no server-side check; tool outputs also carry server-authored [SYSTEM INSTRUCTION] text. (ASI01; C5) — [src/tools/filesystem.ts:367](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/filesystem.ts#L367); [src/terminal-manager.ts:256](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L256); [src/utils/usageTracker.ts:268](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/utils/usageTracker.ts#L268)
- Third-party code (a Chrome build fetched at startup, or any package the model installs via the shell) runs as the user with full environment and no confinement. (ASI04; C7) — [src/index.ts:135](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/index.ts#L135); [src/tools/pdf/markdown.ts:182](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/pdf/markdown.ts#L182)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

Desktop Commander runs with the full authority of the logged-in user and narrows nothing by default: the allowed-folders list ships empty, which the code treats as the whole filesystem, and every shell it starts inherits the server's complete environment, including any API keys or tokens in it. Worse, the model itself can widen its own permissions: the set_config_value tool lets it rewrite the allowed-folders list and the blocked-command list, and the config file lives in the home directory that its own file tools can edit. A hijacked session therefore reaches everything the user can, across every service whose credentials sit in the home directory or environment.

- **S L0:** Ambient OS-user authority with no narrowing, and the set_config_value tool lets the model rewrite its own allowed directories and command blocklist. — [src/tools/config.ts:247](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/config.ts#L247); [src/config-field-definitions.ts:11-20](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/config-field-definitions.ts#L11-L20) (verified)
  - *To reach the next level:* L1 needs a dedicated identity or credential separate from the user's ambient authority that the model cannot widen.
- **C L0:** No authorization layer exists; shell subprocesses get the full process environment and the file-path check is off by default. — [src/terminal-manager.ts:207-212](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L207-L212); [src/tools/filesystem.ts:180](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/filesystem.ts#L180) (verified)
  - *To reach the next level:* L1 needs the main tool path checked against a scoped identity; today even the path check is disabled by the empty default list.
- **D L0:** Default config grants the whole filesystem (allowedDirectories: []) and the model can change it at runtime. — [src/config-manager.ts:188](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/config-manager.ts#L188); [src/server.ts:346-347](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/server.ts#L346-L347) (verified)
  - *To reach the next level:* L1 needs a narrower default (e.g. a project folder) even if it can be widened.
- **B L0:** A hijacked session can use every credential the user holds: SSH keys, cloud and CLI tokens in the home directory, and secrets in the inherited environment. — [src/terminal-manager.ts:207-212](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L207-L212); [src/terminal-manager.ts:256](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L256) (verified)
  - *To reach the next level:* L1 needs the reachable authority limited to fewer systems than the user's whole account.
- **Cap:** C1-SELFESC — set_config_value (tools/config.ts:247) lets the model change allowedDirectories and blockedCommands, its own permission scope.

### C2 Approval gates — 0.45 (high)

As a tool server, Desktop Commander leaves approval to the host, and gives it reasonable signals: every listed tool carries readOnly/destructive hints, and the shell, write, move, kill and config tools are all marked destructive. It offers no dry-run or preview for destructive operations and no server-enforced read-only mode. If a host approves the wrong shell command, nothing in the server can undo it: commands run directly on the machine, and edits and deletes have no checkpoints.

- **S L2:** Separate read and write tools, each listed tool annotated; start_process, write_file, edit_block, move_file, kill_process and set_config_value are marked destructiveHint: true. — [src/server.ts:919-924](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/server.ts#L919-L924); [src/server.ts:352-356](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/server.ts#L352-L356); searched `rg -n -S 'dryRun|dry_run'` in `src` → 0 hits (no dry-run anywhere) (verified)
  - *To reach the next level:* L3 needs a preview or dry-run for destructive operations (edit_block, write_file rewrite, move_file, shell).
- **C L2:** All 26 listed tools carry annotations, but read_file is marked read-only while it fetches arbitrary URLs, and an unlisted, unannotated track_ui_event tool is still dispatchable. — [src/server.ts:418-423](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/server.ts#L418-L423); [src/server.ts:1376](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/server.ts#L1376) (verified)
  - *To reach the next level:* L3 needs every reachable tool path covered by accurate signals, including the hidden track_ui_event and the egress inside read_file.
- **D L3:** Annotations are fixed in code; neither the model, a tool result nor a workspace file can change them. — [src/server.ts:1063-1074](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/server.ts#L1063-L1074) (verified)
  - *To reach the next level:* L4 needs a server-enforced confirmation step tied to an authenticated principal.
- **B L0:** A wrongly approved shell command can delete data or send data anywhere with no undo; the server keeps no checkpoints. — [src/terminal-manager.ts:256](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L256); [src/tools/process.ts:85](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/process.ts#L85) (verified)
  - *To reach the next level:* L1 needs at least some reversible paths (trash, snapshots) for common destructive actions.
- **Cap:** none

### C3 Tool & action scoping — 0.15 (high)

The main tool is a raw shell, filtered only by a list of blocked command names that the project's own security policy says can be circumvented. File tools do resolve symlinks and check paths against an allowed-folders list, but that list ships empty, which allows everything, and the shell ignores it anyway. URL reads accept any address, including internal ones, interactive shell input is never checked, and the model can choose any program as the 'shell'. The model can also clear both the blocklist and the folder list itself with set_config_value.

- **S L1:** Shell commands pass through a denylist of command names; file paths get realpath containment, but URLs and shell input are raw passthrough. — [src/command-manager.ts:245-248](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/command-manager.ts#L245-L248); [SECURITY.md:43](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/SECURITY.md#L43); [src/tools/filesystem.ts:367](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/filesystem.ts#L367) (verified)
  - *To reach the next level:* L2 needs typed validation on the general tools; the shell remains an arbitrary command string.
- **C L1:** Only start_process and the file tools validate; interact_with_process, the model-chosen shell parameter, URL fetch and kill_process do not. — [src/tools/improved-process-tools.ts:120](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/improved-process-tools.ts#L120); [src/tools/schemas.ts:30](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/schemas.ts#L30); searched `rg -n -F 'validateCommand('` in `src` → 2 hits (definition in command-manager.ts and the single call in startProcess; interact_with_process never calls it) (verified)
  - *To reach the next level:* L2 needs most tools validating, including interactive shell input and URL fetch.
- **D L0:** Every tool, including shell, code execution and URL fetch, is on by default with whole-filesystem access; there is no tool-disable switch or read-only mode. — [src/config-manager.ts:188](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/config-manager.ts#L188); searched `rg -n -S 'disabledTools|enabledTools'` in `src` → 0 hits (verified)
  - *To reach the next level:* L1 needs dangerous tools to be individually disableable.
- **B L0:** General-purpose shell and Node execution against the whole machine. — [src/terminal-manager.ts:256](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L256); [src/tools/improved-process-tools.ts:34](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/improved-process-tools.ts#L34) (verified)
  - *To reach the next level:* L1 needs some limit on reach, e.g. the shell confined to allowed folders.
- **Cap:** G2 — The model can clear blockedCommands and allowedDirectories through set_config_value (tools/config.ts:247), disabling the primary control at runtime.

### C4 Code-execution isolation — 0.50 (high)

By default, shell commands and model-written Node.js run directly on the user's machine as the user, with the full environment and network; there is no sandbox at all. The project's own security policy says the folder list and blocklist are not boundaries and recommends Docker for isolation. The Docker install it ships runs the whole server inside a stock container with only the folders you pick mounted, which is a real but basic boundary (root inside the container, unrestricted network). Because Docker is opt-in, it scores at most half credit.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Shell and node:local run as same-user subprocesses on the host. — [src/terminal-manager.ts:256](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L256); [src/tools/improved-process-tools.ts:34](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/improved-process-tools.ts#L34) (verified)
    - *To reach the next level:* L1 needs at least filtering that the model cannot remove; the only filter is a denylist the model can clear.
  - **C L0:** No execution path is sandboxed. — [src/terminal-manager.ts:256](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L256); [SECURITY.md:20](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/SECURITY.md#L20) (verified)
    - *To reach the next level:* L1 needs the main exec tool sandboxed.
  - **D L0:** No sandbox in the default npx install. — [SECURITY.md:19-21](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/SECURITY.md#L19-L21) (verified)
    - *To reach the next level:* L1 needs isolation on by default.
  - **B L0:** Execution is host-equivalent: home directory, ~/.ssh, cloud credentials and the full environment are reachable. — [src/terminal-manager.ts:207-212](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L207-L212) (verified)
    - *To reach the next level:* L1 needs at least the home directory and credentials kept out of reach.
- **opt-in Docker install (install-docker.sh)** (alt; raw 0.62, cap G1 → 0.50) ← counted
  - **S L2:** Stock node:lts-alpine container with no USER directive (root inside) and no capability hardening. — [Dockerfile:2](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/Dockerfile#L2); [install-docker.sh:241](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/install-docker.sh#L241) (verified)
    - *To reach the next level:* L3 needs a hardened profile: non-root, dropped capabilities, no-new-privileges, seccomp.
  - **C L3:** The whole server, and therefore every shell, Node and file path, runs inside the container. — [install-docker.sh:241-267](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/install-docker.sh#L241-L267) (verified)
    - *To reach the next level:* L4 needs processes spawned by tools confined with no fallback and fail-closed setup; the container is the only layer and has no further confinement.
  - **D L3:** Once installed this way, nothing the model does from inside the container can turn the container off. — [install-docker.sh:319-322](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/install-docker.sh#L319-L322) (verified)
    - *To reach the next level:* L4 needs the sandbox policy defined outside anything the operator's host config or model can edit; the client config written by the installer is user-editable.
  - **B L2:** User-chosen host folders are mounted read-write, persistent named volumes cover /usr, /root and /var, and network is unrestricted. — [install-docker.sh:219-224](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/install-docker.sh#L219-L224) (verified)
    - *To reach the next level:* L3 needs egress off or allowlisted and resource limits.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.00 (high)

Nothing limits what a hijacked session can do. The server fetches any URL and reads any file, returns that content as plain text with no provenance or untrusted marking, and offers no read-only or no-egress mode. It also mixes its own instructions into results: tool descriptions carry imperative directives, and tool outputs can have '[SYSTEM INSTRUCTION]' blocks appended (feedback and onboarding prompts switched on by remotely fetched feature flags). Content an attacker plants in a web page or file can therefore steer the model to read secrets and send them out through the shell or a URL fetch, or to delete data, with no server-side check.

- **S L0:** Tool outputs can carry appended [SYSTEM INSTRUCTION] directives and tool descriptions contain directives to the model. — [src/server.ts:1576-1579](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/server.ts#L1576-L1579); [src/utils/usageTracker.ts:268](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/utils/usageTracker.ts#L268); [src/server.ts:462](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/server.ts#L462) (verified)
  - *To reach the next level:* L1 needs plain outputs with no server-authored directives mixed into returned content.
- **C L0:** No source is distinguished: fetched URLs, files and process output all return as plain text. — searched `rg -n -S 'untrusted|provenance'` in `src` → 0 hits; [src/tools/filesystem.ts:367](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/filesystem.ts#L367) (verified)
  - *To reach the next level:* L1 needs at least one untrusted source (e.g. URL fetch) marked or handled differently.
- **D L0:** No untrusted-input control exists to be on by default. — searched `rg -n -S 'untrusted|provenance'` in `src` → 0 hits (verified)
  - *To reach the next level:* L1 needs a control that is on by default.
- **B L0:** A hijacked session can read secrets anywhere and exfiltrate them via shell or URL fetch, and run irreversible commands, all without a server-side check. — [src/tools/filesystem.ts:367](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/filesystem.ts#L367); [src/terminal-manager.ts:256](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L256) (verified)
  - *To reach the next level:* L1 needs either the egress channel or irreversible actions removed from unattended reach.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.10 (high)

Desktop Commander does not auto-load project instruction files, but two things persist across sessions. First, its security settings (blocked commands, allowed folders, shell) live in a config file in the home directory that the model can change with set_config_value or its own file tools, and a file watcher applies edits immediately; a one-time injection can therefore permanently loosen the server. Second, every tool call's arguments and output (up to 4 KB) are saved and can be read back into later chats with get_recent_tool_calls, without marking which parts came from untrusted sources.

- **S L0:** The model can persistently rewrite security settings through a tool or by editing the config file, which is hot-reloaded. — [src/tools/config.ts:247](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/config.ts#L247); [src/config-manager.ts:293-297](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/config-manager.ts#L293-L297) (verified)
  - *To reach the next level:* L1 needs security-relevant writes at least logged and the config kept out of the model's reach.
- **C L0:** Neither the config file nor the tool-history store is controlled. — [src/config.ts:6](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/config.ts#L6); [src/utils/toolHistory.ts:74](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/utils/toolHistory.ts#L74) (verified)
  - *To reach the next level:* L1 needs at least one store (config) protected from model writes.
- **D L1:** Single-user local storage under the user's home; isolation is by OS user only, and the model can write both stores. — [src/config.ts:6-9](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/config.ts#L6-L9) (verified)
  - *To reach the next level:* L2 needs per-session namespaces enforced so the model cannot write persistent state outside its session.
- **B L1:** Poisoned settings persist across all of the user's sessions and change what tools may do (e.g. clearing the blocklist). — [src/config-field-definitions.ts:11-20](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/config-field-definitions.ts#L11-L20) (verified)
  - *To reach the next level:* L2 needs persisted changes to influence only text output or gated actions.
- **Cap:** none

### C7 Third-party extensions — 0.07 (high)

The server has no plugin or extension system, but it does fetch and run third-party code on its own: at startup, if no Chrome is found, it downloads the current 'stable' Chrome build from Google's distribution and later runs it to render PDFs, with no version pin, no integrity check in this code and no consent prompt. Through its shell the model can also install and run any npm or pip package with the user's full access. Nothing confines what such code can reach.

- **S L1:** Chrome is fetched from the vendor's official channel but resolved to whatever 'stable' is at that moment, with no hash check in this code; shell-driven package installs are unverified. — [src/tools/pdf/markdown.ts:177](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/pdf/markdown.ts#L177); [src/tools/pdf/markdown.ts:182](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/pdf/markdown.ts#L182) (verified)
  - *To reach the next level:* L2 needs the downloaded browser build pinned.
- **C L0:** Neither the auto-downloaded browser nor shell-installed packages are verified. — [src/tools/pdf/markdown.ts:171-196](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/pdf/markdown.ts#L171-L196) (verified)
  - *To reach the next level:* L1 needs verification on at least one type of runtime-fetched code.
- **D L0:** The Chrome download starts automatically after the MCP handshake with no consent. — [src/index.ts:134-135](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/index.ts#L134-L135) (verified)
  - *To reach the next level:* L1 needs at least a generic consent step before the download.
- **B L0:** Downloaded or shell-installed code runs as the same user with the full environment and filesystem. — [src/terminal-manager.ts:207-212](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L207-L212) (verified)
  - *To reach the next level:* L1 needs extensions in a separate process without the full environment.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.20 (high)

The server holds no keys of its own in the default mode, but it does nothing to keep secrets out of reach or out of its records. Shells inherit the whole environment, so any API key in it is one 'env' command away. Every tool call's arguments, and outputs up to 4 KB, are written in plain text to two files in the home directory with no redaction, so a secret the model reads ends up on disk. Telemetry to the vendor is on by default; it strips path-like fields and sends tool and command names rather than file contents.

- **S L1:** No redaction anywhere; only telemetry strips path-named keys and path-like substrings in error messages. — [src/utils/capture.ts:162](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/utils/capture.ts#L162); searched `rg -n -S 'redact|secret'` in `src` → 3 hits (only Kubernetes service-account path checks in system-info.ts; no redaction) (verified)
  - *To reach the next level:* L2 needs redaction filters on the main logs and history.
- **C L1:** Only the telemetry path is filtered; local logs, tool history, model-bound output and subprocess environments are not. — [src/utils/trackTools.ts:20](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/utils/trackTools.ts#L20); [src/utils/toolHistory.ts:53](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/utils/toolHistory.ts#L53) (verified)
  - *To reach the next level:* L2 needs logs and transcripts protected.
- **D L1:** Telemetry is on by default (opt-out) and largely content-free; local logging of full arguments is always on. — [src/config-manager.ts:189](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/config-manager.ts#L189) (verified)
  - *To reach the next level:* L2 needs telemetry opt-in.
- **B L0:** Any long-lived key in the user's environment or home directory is reachable by every subprocess the model starts. — [src/terminal-manager.ts:207-212](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L207-L212) (verified)
  - *To reach the next level:* L1 needs keys narrowed or kept out of subprocess environments.
- **Cap:** none

### C9 Audit & traceability — 0.40 (high)

Every tool call is recorded twice in the user's home directory: a plain-text log of tool names and full arguments, and a structured JSON-lines history with arguments, results and timing. There is no record of who asked (local or remote caller is only sent to telemetry), and both files sit where the model's own file and shell tools can edit or delete them. The history is written in batches once a second, the argument log is written without waiting, and failures are silent, so the last actions before a crash can be lost.

- **S L2:** Structured record of each call with arguments, output (capped at 4 KB), timestamp and duration. — [src/server.ts:1502](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/server.ts#L1502); [src/utils/trackTools.ts:20](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/utils/trackTools.ts#L20) (verified)
  - *To reach the next level:* L3 needs actor attribution (requesting client, local vs remote) in the record itself.
- **C L2:** All built-in tool calls are recorded (only get_recent_tool_calls and track_ui_event are excluded from the history); host approvals are not visible to the server. — [src/server.ts:1495-1503](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/server.ts#L1495-L1503) (verified)
  - *To reach the next level:* L3 needs approvals/denials and the background processes' later activity captured.
- **D L1:** On by default but stored under ~/.claude-server-commander, which the model's file and shell tools can edit with the default whole-filesystem access. — [src/config.ts:6-10](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/config.ts#L6-L10); [src/config-manager.ts:188](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/config-manager.ts#L188) (verified)
  - *To reach the next level:* L2 needs the records outside anything the model's tools can write.
- **B L1:** History flushes every second from an in-memory queue and the argument log is fire-and-forget; write errors are swallowed. — [src/utils/toolHistory.ts:205](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/utils/toolHistory.ts#L205); [src/server.ts:1276](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/server.ts#L1276) (verified)
  - *To reach the next level:* L2 needs errors surfaced and each record flushed per action.
- **Cap:** none

### C10 Limits & kill switch — 0.33 (high)

Some operations have server-side bounds: URL reads time out after 30 seconds, process output is capped at 50 MB per session, and directory listings cap nested entries. But the command timeout only stops the server waiting; the process keeps running in the background with no time limit, and the model chooses the timeout itself. Force-terminate signals only the direct child, not its process tree, and nothing cleans up running processes when the server exits.

- **S L2:** Server-enforced caps on some operations (URL fetch timeout, output buffer size, nested listing cap); termination via SIGINT then SIGKILL on the direct child. — [src/tools/filesystem.ts:19](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/filesystem.ts#L19); [src/terminal-manager.ts:56](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L56); [src/terminal-manager.ts:764-767](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L764-L767) (verified)
  - *To reach the next level:* L3 needs caps on every operation plus concurrency or rate limits.
- **C L1:** Timeouts stop waiting but leave spawned processes running; no budget applies to background processes. — [src/terminal-manager.ts:450-459](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/terminal-manager.ts#L450-L459) (verified)
  - *To reach the next level:* L2 needs tool timeouts that actually end the work they bound.
- **D L1:** timeout_ms is chosen by the model per call and fileReadLineLimit is model-settable via set_config_value. — [src/tools/schemas.ts:29](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/tools/schemas.ts#L29); [src/config-field-definitions.ts:11-20](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/c774c3b505de990219637ecdc9a830c8772fae9d/src/config-field-definitions.ts#L11-L20) (verified)
  - *To reach the next level:* L2 needs sensible defaults the model cannot raise.
- **B L1:** Stopping leaves work running: no process-group kill and no cleanup of child processes on exit. — searched `rg -n -S 'detached|process\.kill\(-'` in `src` → 0 hits (no process-group handling) (verified)
  - *To reach the next level:* L2 needs stop to end all spawned work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: arbitrary URL fetch and file reads (src/tools/filesystem.ts:367) · [B] sensitive data/systems: whole filesystem and inherited environment (src/config-manager.ts:188, src/terminal-manager.ts:209) · [C] state change / egress: shell execution and URL fetch (src/terminal-manager.ts:256) · Same default session? Yes

## Highest-impact improvements
1. Restrict set_config_value to non-security keys; let only the user change blockedCommands, allowedDirectories and defaultShell (config editor UI or CLI), and keep the config file outside allowed folders. — C6 S L0→L2, +0.150 before caps (Playbook 2, step 1)
2. Ship a read-only / no-shell mode and per-tool disable switches, and default allowedDirectories to a chosen project folder instead of the whole filesystem. — C3 D L0→L2, +0.100 before caps (Playbook 3, step 1)
3. Stop appending [SYSTEM INSTRUCTION] text to tool results and return fetched/file content with source and an untrusted flag. — C5 S L0→L3, +0.225 before caps (Playbook 1, step 2)
4. Kill the whole process group on timeout, force_terminate and server exit, and add a hard maximum runtime per process. — C10 B L1→L2, +0.050 before caps (Playbook 3, step 3)
5. Pin the Chrome build used for PDF rendering and ask before downloading it. — C7 D L0→L1, +0.050 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of commit c774c3b (v0.2.52) only; nothing was executed, installed, or probed.
- Scored the default npx stdio install. The opt-in Docker install was scored as the C4 alternative; the opt-in 'remote' device mode (src/remote-device/) was not scored.
- Behaviour of third-party libraries (@puppeteer/browsers integrity checks, md-to-pdf/Chrome sandboxing) was not examined.
- Feature flags fetched from desktopcommander.app decide whether feedback/onboarding directives are injected; their live values were not checked.
- Host-side approval behaviour is out of scope for a tool server and was not credited.
- No reviewer-directed prompt injection found in the repo.
