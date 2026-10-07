# Defense-in-Depth Score: MCP Filesystem server

**Repo:** https://github.com/modelcontextprotocol/servers (`src/filesystem`) · **Commit:** `f46d9578190b476b3501923ea8977d899e8db2cb` · **Reviewed:** 2026-10-03
**What it is:** Reference MCP server for read/write filesystem access within allowed dirs
**Category:** AI Assistants
**Scored configuration:** stdio server launched via npx with one or more allowed directories on the command line (package @modelcontextprotocol/server-filesystem 0.6.3 per package.json), host client may supply MCP roots; Docker image footnoted.
**Agent surface (default):** code execution no · filesystem write yes · network egress no · external credentials no · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 5.3 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C2 | Approval gates | L2 | L3 | L3 | L1 | 0.57 | — | **0.57** | High |
| C3 | Tool & action scoping | L3 | L3 | L1 | L2 | 0.60 | — | **0.60** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L1 | 0.12 | — | **0.12** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L4 | 0.45 | — | **0.45** | High |
| C9 | Audit & traceability | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C10 | Limits & kill switch | L1 | L1 | L0 | L1 | 0.20 | G1 | **0.20** | High |

Controls where a risk surface exists: 2.33 / 7.0 (33%); 3 criteria scored SA (surface absent).

A small, well-fenced file server: every path is resolved through symlinks and checked against the directories you allow, and it never runs code, touches the network or holds credentials. The main risk is what it hands a hijacked model inside those directories: write, edit and move tools are always on with no read-only switch and no undo, and file contents (including any .env or key files there) go to the model unmarked. It keeps no log of what it did and puts no limits on file sizes or recursive listings, so allow only the narrowest directories you need and rely on the host to approve writes.

## Critical gaps
- None: no criterion is capped and no blast radius is at the worst level.

## Criterion details

### C1 Identity & least privilege — 0.38 (high)

The server holds no credentials, reads no environment variables, starts no other programs and makes no network calls, so there is no token to steal or forward. It does, however, run with the full file permissions of whoever launched it (or as root inside the shipped Docker image), and the only thing narrowing that is the allowed-directories check, which is scored under tool scoping. If that check failed, the server could read and write anything the user can, including SSH keys, cloud credential files and shell startup files.

- **S L1:** No credential chain, token, or subprocess environment is ever used; the process's only authority is the launching OS user's filesystem permissions (root in the Docker image, which has no USER directive), narrowed solely by the path allowlist credited in C3. — searched `rg -n -S 'process\.env|token|secret|password|apikey|api_key|credential'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (The server reads no environment variables and handles no credentials.); searched `rg -n -S 'child_process|spawn\(|exec\(|execFile|\beval\(|new Function'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No process spawning, shell, or dynamic code evaluation anywhere in the server source.); [src/filesystem/Dockerfile:11-25](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/Dockerfile#L11-L25) (No USER directive: the container process runs as root.) (verified)
  - *To reach the next level:* No dedicated low-privilege identity (e.g. a non-root container user or OS-level read-only handles for read tools).
- **C L2:** Every tool executes in the same process with the same OS identity; there is no subprocess, extension, or network path that could pick up broader authority. — [src/filesystem/index.ts:189-214](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L189-L214); searched `rg -n -S 'child_process|spawn\(|exec\(|execFile|\beval\(|new Function'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No process spawning, shell, or dynamic code evaluation anywhere in the server source.); searched `rg -n -S 'fetch\(|https?\.|net\.|axios|WebSocket|dgram'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No network client; the only transport is stdio.) (verified)
  - *To reach the next level:* No per-tool narrower identity (read tools share the write-capable process authority).
- **D L2:** Defaults are operator-chosen directories with full read/write; MCP client roots silently replace the command-line directories entirely, which can widen scope without warning. — [src/filesystem/index.ts:725-734](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L725-L734); [src/filesystem/README.md:29](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/README.md#L29) (verified)
  - *To reach the next level:* No read-only default; widening via client roots is not restricted to a subset of the operator's command-line directories and is only logged to stderr.
- **B L1:** If the path authorization fails, the server can read and write every file the OS user can (credential files, shell rc files, SSH authorized_keys); it has no exec or egress of its own, so further use of that access needs the host or a later process. — [src/filesystem/lib.ts:201-203](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/lib.ts#L201-L203); [src/filesystem/lib.ts:205-209](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/lib.ts#L205-L209); searched `rg -n -S 'fetch\(|https?\.|net\.|axios|WebSocket|dgram'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No network client; the only transport is stdio.) (verified)
  - *To reach the next level:* Write access is not confined to a single non-sensitive scope independent of the path check (e.g. by an OS sandbox or container mount).
- **Cap:** none
- **Notes:** Docker mode (documented alternative) narrows reach to explicitly mounted directories, with ro mounts available, but the process is root inside the container.

### C2 Approval gates — 0.57 (high)

As a tool server, this project cannot ask the user for approval itself; it gives the host the information to do so. Reading tools and writing tools are separate, and every tool carries accurate hints: all ten read tools are marked read-only and the write, edit and move tools are marked destructive. The edit tool offers a dry-run preview, but overwriting a whole file and moving files have no preview, and the server has no read-only mode of its own. Overwrites are immediate and keep no backup, so a wrongly approved write cannot be undone by the server.

- **S L2:** Separate read and write tools with readOnlyHint/destructiveHint on every one of the 14 tools, matching what each handler does; dry-run exists only for edit_file. — [src/filesystem/index.ts:221](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L221); [src/filesystem/index.ts:371](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L371); [src/filesystem/index.ts:401](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L401); [src/filesystem/index.ts:398](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L398); [src/filesystem/lib.ts:336](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/lib.ts#L336) (verified)
  - *To reach the next level:* write_file and move_file have no preview/dry-run, so not every destructive operation can be previewed.
- **C L3:** All 14 registered tools carry annotations, and every tool marked read-only only calls read/stat/readdir APIs, so no mutating path is hidden behind a read-only hint. — searched `rg -n 'readOnlyHint: true'` in `src/filesystem/index.ts` → 10 hits (Ten read tools, each verified to perform only reads.); searched `rg -n 'readOnlyHint: false'` in `src/filesystem/index.ts` → 4 hits (write_file, edit_file, create_directory, move_file.); [src/filesystem/index.ts:426](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L426); [src/filesystem/index.ts:630](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L630) (verified)
  - *To reach the next level:* No server-side mechanism that rejects or holds mutating calls; coverage depends on the host honouring the hints.
- **D L3:** Annotations are hard-coded in the tool registrations and cannot be changed by the model, tool input, or any file in the allowed directories; there are no persisted allow rules in the server. — [src/filesystem/index.ts:371](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L371); searched `rg -n -S 'process\.env|token|secret|password|apikey|api_key|credential'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (The server reads no environment variables and handles no credentials.) (verified)
  - *To reach the next level:* No server-enforced confirmation step tied to an authenticated principal, and no time-bounded elevation.
- **B L1:** write_file and edit_file replace files in place via temp-file rename with no backup or undo; move_file refuses to overwrite and there is no delete tool, so some operations are low-impact. — [src/filesystem/lib.ts:215-219](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/lib.ts#L215-L219); [src/filesystem/lib.ts:246-255](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/lib.ts#L246-L255) (verified)
  - *To reach the next level:* No checkpoint, backup, or trash for overwritten files, so the common write case is not reversible.
- **Cap:** none

### C3 Tool & action scoping — 0.60 (high)

This is the server's strongest area. Every tool takes a path and every path goes through one validation function that resolves symlinks with realpath, checks containment against the allowed directories with a separator-aware comparison, rejects null bytes, and for new files walks each existing parent component checking it stays inside. Writes avoid following pre-existing symlinks by using exclusive creation and atomic rename. The tools are narrow (no shell, no network), but all of them, including write, edit and move, are always on, there is no read-only switch, and nothing bounds file sizes or result counts. Path containment is not a complete boundary.

- **S L3:** Resolved-path containment: normalize + startsWith(dir + sep), realpath re-check for existing targets, per-component realpath walk for not-yet-existing paths, null-byte rejection, exclusive-create writes. — [src/filesystem/lib.ts:154-158](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/lib.ts#L154-L158); [src/filesystem/lib.ts:162-168](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/lib.ts#L162-L168); [src/filesystem/lib.ts:131-134](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/lib.ts#L131-L134); [src/filesystem/path-validation.ts:84](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/path-validation.ts#L84); [src/filesystem/path-validation.ts:23](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/path-validation.ts#L23); [src/filesystem/lib.ts:209](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/lib.ts#L209) (verified)
  - *To reach the next level:* Containment enforcement is not a complete boundary.
- **C L3:** Every tool handler calls validatePath on each path argument (13 call sites covering all path-taking tools), and recursive search/tree re-validate every entry; there are no extension tools. — searched `rg -n 'validatePath\('` in `src/filesystem/index.ts` → 13 hits (One call per path argument across all path-taking tools; list_allowed_directories takes no path.); [src/filesystem/lib.ts:458](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/lib.ts#L458); [src/filesystem/index.ts:571](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L571) (verified)
  - *To reach the next level:* Validation is called by hand in each handler rather than by a central layer that new tools inherit automatically.
- **D L1:** All 14 tools, including write_file, edit_file, move_file and create_directory, are registered unconditionally; there is no exec or network tool, but there is no server flag to disable writes (read-only only via Docker ro mounts, documented in the README). — [src/filesystem/index.ts:358-372](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L358-L372); [src/filesystem/index.ts:33-40](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L33-L40); [src/filesystem/README.md:215](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/README.md#L215) (verified)
  - *To reach the next level:* Write tools cannot be disabled individually or by a read-only mode in the server.
- **B L2:** Misuse is confined to the allowed directories but with full create/overwrite/move rights inside them and no quantity bounds. — [src/filesystem/lib.ts:201-203](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/lib.ts#L201-L203); [src/filesystem/index.ts:430](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L430) (verified)
  - *To reach the next level:* No quantity bounds (bytes written, files touched, read sizes).
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

The server never runs code: there is no shell, no process spawning and no dynamic evaluation anywhere in its source. It can write files that some other program might later execute, but that execution happens outside this server and is not counted here.

- **Structural absence:** searched `rg -n -S 'child_process|spawn\(|exec\(|execFile|\beval\(|new Function'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No process spawning, shell, or dynamic code evaluation anywhere in the server source.)
- **Notes:** Files written into allowed directories (scripts, git hooks, editor settings) may be executed by other tools; that is a consequence of C3/C5 scope, not an execution path in this server.

### C5 Untrusted input blast radius — 0.12 (high)

The server's job is to hand file contents to the model, so any file in the allowed directories can carry a prompt injection. Contents are returned as plain text with no marker that they are untrusted and no source path attached (except in the multi-file read), so the host gets nothing it could act on. The server has no network access of its own, so it cannot itself send data out, but a hijacked model can use it to overwrite or rearrange files without any check from the server, and paired with any egress-capable tool on the same host the full leak-plus-damage chain is open.

- **S L1:** Outputs are plain text blocks with a structuredContent field holding only the same text; no provenance or untrusted flag, and no read-only/no-egress mode. — [src/filesystem/index.ts:208-211](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L208-L211); [src/filesystem/index.ts:343](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L343); searched `rg -n -S 'untrusted|provenance'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No provenance or untrusted marking anywhere.) (verified)
  - *To reach the next level:* Structured outputs do not separate returned content from metadata such as the source path.
- **C L0:** No source is distinguished: every read tool returns file content with the same standing. — [src/filesystem/index.ts:337-354](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L337-L354) (verified)
  - *To reach the next level:* No read tool marks its output as untrusted data.
- **D L0:** There is no injection-related mechanism to be on or off. — searched `rg -n -S 'untrusted|provenance'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (verified)
  - *To reach the next level:* No default-on provenance or untrusted marking exists.
- **B L1:** A hijacked host model can overwrite, edit and move any file in the allowed directories with no server-side check, but the server itself has no egress channel. — [src/filesystem/index.ts:373-381](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L373-L381); searched `rg -n -S 'fetch\(|https?\.|net\.|axios|WebSocket|dgram'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No network client; the only transport is stdio.) (verified)
  - *To reach the next level:* Irreversible writes are not gated or disabled by the server once untrusted content has been read.
- **Cap:** none
- **Notes:** S is L1 only because outputs are plain (the anchor for no directives); no opt-in mechanism exists, so G1 does not apply and D is L0 for absence.

### C6 Memory, context & configuration integrity — 1.00 (high)

Nothing the model does can change how the server behaves later. There is no memory, cache or database, no settings file is read from the allowed directories, and configuration comes only from command-line arguments and the client's roots list.

- **Structural absence:** searched `rg -n -S 'dotenv|persist|AGENTS\.md|CLAUDE\.md|sqlite|localStorage|memory'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 1 hits (Single hit is a code comment ('memory-efficient') at index.ts:172, not a store.); searched `rg -n -S 'process\.env|token|secret|password|apikey|api_key|credential'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (The server reads no environment variables and handles no credentials.)

### C7 Third-party extensions — 1.00 (high)

The server loads no third-party code at runtime: no plugins, no dynamic imports, no package installs and no other MCP servers. How the host installs this server (the README's npx -y line fetches the latest version) is the host's supply-chain concern and is not scored here.

- **Structural absence:** searched `rg -n -S 'import\(|require\(|plugin|npx'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No dynamic imports, plugin loading, or package launching.); searched `rg -n -S 'child_process|spawn\(|exec\(|execFile|\beval\(|new Function'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No process spawning, shell, or dynamic code evaluation anywhere in the server source.)

### C8 Secrets & sensitive-data protection — 0.45 (high)

The server holds no keys or tokens and reads no environment variables, so there is nothing of its own to leak, and its stderr log lines contain directory paths and error messages but never file contents. It has no telemetry. But it does nothing to protect secrets that sit in the allowed directories: a .env file, private key or credentials file is returned to the model verbatim, with no denylist or redaction.

- **S L1:** No secret material is held; logs carry only paths/counts/errors, but there is no redaction or sensitive-file denylist for content returned to the model. — searched `rg -n -S 'process\.env|token|secret|password|apikey|api_key|credential'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (The server reads no environment variables and handles no credentials.); [src/filesystem/index.ts:730](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L730); [src/filesystem/lib.ts:201-203](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/lib.ts#L201-L203) (verified)
  - *To reach the next level:* No type-level masking, log filters, or redaction of secret-looking content in tool results.
- **C L1:** Only the log path is effectively content-free; model-bound tool results and error messages carry file contents and full paths unfiltered. — [src/filesystem/index.ts:343-346](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L343-L346); searched `rg -n -S 'appendFile|audit|sendLoggingMessage'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No audit file, no MCP logging notifications.) (verified)
  - *To reach the next level:* Model-bound messages are not covered by any protection.
- **D L2:** No telemetry or crash reporting exists and logging is minimal by default; there is no redaction to enable. — searched `rg -n -S 'fetch\(|https?\.|net\.|axios|WebSocket|dgram'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No network client; the only transport is stdio.); searched `rg -n -S 'appendFile|audit|sendLoggingMessage'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No audit file, no MCP logging notifications.) (verified)
  - *To reach the next level:* No always-on redaction.
- **B L4:** Ambient-only design: the server accepts and stores no key material, so a leak of the server exposes no credential of its own; residual gap is workspace secrets returned verbatim (footnoted). — searched `rg -n -S 'process\.env|token|secret|password|apikey|api_key|credential'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No credential handling in the server.); [src/filesystem/index.ts:33](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L33) (verified)
- **Cap:** none
- **Notes:** Residual: secret files inside allowed directories (.env, keys) are readable and returned to the model unredacted.

### C9 Audit & traceability — 0.00 (high)

The server keeps no record of what it did. Tool calls, their arguments and results are not logged anywhere; the only output is a handful of startup and roots-change messages on stderr. Reconstructing an incident depends entirely on whatever the host application chose to log.

- **S L0:** No tool call is recorded; stderr messages cover only startup, directory validation and roots updates. — searched `rg -n -S 'appendFile|audit|sendLoggingMessage'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No audit file, no MCP logging notifications.); [src/filesystem/index.ts:777](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L777) (verified)
  - *To reach the next level:* No structured record of tool calls with arguments, status and timestamps.
- **C L0:** Nothing about tool execution is recorded. — searched `rg -n -S 'appendFile|audit|sendLoggingMessage'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No audit file, no MCP logging notifications.) (verified)
  - *To reach the next level:* No tool path is recorded.
- **D L0:** There is no audit record to enable. — searched `rg -n -S 'appendFile|audit|sendLoggingMessage'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No audit file, no MCP logging notifications.) (verified)
  - *To reach the next level:* No default-on record.
- **B L0:** With no record, actions leave no trace in the server. — searched `rg -n -S 'appendFile|audit|sendLoggingMessage'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No audit file, no MCP logging notifications.) (verified)
  - *To reach the next level:* No record exists to flush or surface failures for.
- **Cap:** none

### C10 Limits & kill switch — 0.20 (high)

The server puts no limits on its own work. Whole files are read into memory regardless of size, the multi-file read takes any number of paths at once, and the directory tree and search tools walk the entire directory with no depth or result limit. The only bounds are the optional head and tail line counts the caller may pass. Requests cannot be cancelled mid-way, though stopping the server process ends everything because it starts no background work or child processes.

- **S L1:** Only caller-chosen head/tail line limits on read_text_file; no server-enforced caps or cancellation. — [src/filesystem/index.ts:195-206](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L195-L206); searched `rg -n -S 'signal|AbortSignal|timeout|setTimeout|maxSize|MAX_|limit'` in `src/filesystem/index.ts src/filesystem/lib.ts` → 0 hits (No timeouts, size caps, abort handling, or rate limits in the server source.) (verified)
  - *To reach the next level:* No server-enforced cap on output size, recursion depth or timeouts.
- **C L1:** The optional limit applies to one tool; read_multiple_files, read_media_file, directory_tree and search_files are unbounded. — [src/filesystem/index.ts:338-349](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L338-L349); [src/filesystem/lib.ts:451-483](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/lib.ts#L451-L483); [src/filesystem/index.ts:174-187](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L174-L187) (verified)
  - *To reach the next level:* Limits do not cover the recursive and multi-file tools.
- **D L0:** Unlimited by default. — searched `rg -n -S 'signal|AbortSignal|timeout|setTimeout|maxSize|MAX_|limit'` in `src/filesystem/index.ts src/filesystem/lib.ts` → 0 hits (No timeouts, size caps, abort handling, or rate limits in the server source.) (verified)
  - *To reach the next level:* No default limits exist.
- **B L1:** No ceilings, but each call is bounded by the finite filesystem, recursion does not follow symlinked directories, and killing the stdio process stops all work with nothing orphaned. — [src/filesystem/index.ts:595-598](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/filesystem/index.ts#L595-L598); searched `rg -n -S 'child_process|spawn\(|exec\(|execFile|\beval\(|new Function'` in `src/filesystem/index.ts src/filesystem/lib.ts src/filesystem/path-utils.ts src/filesystem/path-validation.ts src/filesystem/roots-utils.ts` → 0 hits (No process spawning, shell, or dynamic code evaluation anywhere in the server source.) (verified)
  - *To reach the next level:* No per-call time or size ceiling, so a single request can exhaust memory before it completes.
- **Cap:** G1 — The only bound (head/tail line counts) is opt-in per call; nothing is limited by default.

## Rule-of-Two check
[A] untrusted input: File contents from allowed directories returned verbatim to the model (src/filesystem/index.ts:208) · [B] sensitive data/systems: Any file in allowed directories, including .env/key files, read unfiltered (src/filesystem/lib.ts:202) · [C] state change / egress: write_file/edit_file/move_file overwrite and move files without server-side gating (src/filesystem/index.ts:375); no egress of its own · Same default session? Yes

## Highest-impact improvements
1. Add a --read-only flag (or require --allow-write) that skips registering write_file, edit_file, move_file and create_directory. — C3 D L1→L3, +0.100 before caps (Playbook 3)
2. Write a structured JSON-lines record of every tool call (tool, arguments, result status, timestamp) outside the allowed directories, or emit MCP logging notifications. — C9 S L0→L2, +0.150 before caps (Playbook 1 step 3)
3. Enforce server-side caps: maximum file size read, maximum paths per read_multiple_files, maximum depth/entries for directory_tree and search_files, and honour request cancellation. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
4. Return source path and an untrusted-content flag in structuredContent for every read tool. — C5 S L1→L3, +0.150 before caps (Playbook 1)
5. Add dryRun previews to write_file (diff against existing content) and move_file. — C2 S L2→L3, +0.075 before caps (Playbook 5)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Only the src/filesystem subpath was reviewed; the MCP TypeScript SDK (transport, schema validation, cancellation plumbing) and the hosts that launch this server were not examined.
- Windows-specific path handling (drive letters, UNC paths) was read but not analysed in depth; ratings assume a POSIX host.
- Tool servers do not own the approval gate or model context; C2 and C5 rate only what the server gives the host, not any host's behaviour.
- README claims (e.g. 'Create/list/delete directories') were checked against code: no delete tool exists. No text aimed at AI reviewers was found in the subpath.
