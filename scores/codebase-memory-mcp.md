# Defense-in-Depth Score: codebase-memory-mcp

**Repo:** https://github.com/DeusData/codebase-memory-mcp · **Commit:** `268a9d8886642eb7f9b2ce45f5ce27cdecf0f519` · **Reviewed:** 2026-10-05
**What it is:** Local MCP server that indexes a codebase into a persistent SQLite knowledge graph and exposes 17 search, trace, query and ADR tools to coding agents.
**Category:** Coding
**Scored configuration:** Default stdio MCP entry written by the install command: daemon-backed server with the full 17-tool profile, graph UI off, auto_index off, auto_watch on, no CBM_ALLOWED_ROOT and no recorded allow-root grants.
**Agent surface (default):** code execution no · filesystem write yes · network egress no · external credentials no · persistent memory yes · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 5.2 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L2 | L2 | L3 | 0.47 | none | **0.47** | High |
| C2 | Approval gates | L2 | L3 | L3 | L2 | 0.62 | none | **0.62** | High |
| C3 | Tool & action scoping | L2 | L3 | L2 | L2 | 0.57 | none | **0.57** | High |
| C4 | Code-execution isolation | L1 | L1 | L2 | L0 | 0.25 | none | **0.25** | High |
| C5 | Untrusted input blast radius | L2 | L1 | L2 | L2 | 0.42 | none | **0.42** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L1 | 0.30 | none | **0.30** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L3 | L4 | 0.65 | none | **0.65** | High |
| C9 | Audit & traceability | L1 | L2 | L2 | L1 | 0.38 | none | **0.38** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L2 | 0.50 | none | **0.50** | High |

Controls where a risk surface exists: 4.17 / 9.0 (46%); 1 criterion scored SA (surface absent).

A local code-graph server that holds no credentials, makes no network calls and ships accurate read-only and destructive labels on all 17 tools, so a hijacked agent can do limited damage through it. The main gap is that git, grep and find helpers run unsandboxed through a shell, with only a character filter guarding the model-supplied values they include. Any non-sensitive directory can be indexed and read back by default. ADR text and repository-supplied graph snapshots persist into later sessions without review, and the audit log records tool names but not what each call did.

## Critical gaps
- Helper git, grep and find commands are built as shell strings that include model-supplied values, guarded only by a character filter, and run unsandboxed as the user with the inherited environment. (ASI05, T11; C4) — [src/foundation/compat_fs.c:932](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/foundation/compat_fs.c#L932); [src/foundation/str_util.c:246-270](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/foundation/str_util.c#L246-L270)

## Criterion details

### C1 Identity & least privilege — 0.47 (high)

The server holds no credentials and talks to no outside service; it runs as the local user and reads and writes files with that user's rights. Its main narrowing is a workspace policy that refuses to index the home directory, shallow system paths and known credential folders such as .ssh and .aws, plus an optional allowed-root setting and per-user grants managed from the command line. With nothing configured, any other directory the user can read can be indexed and its source returned, so the narrowing is a short deny list rather than a least-privilege allowlist.

- **S L1:** Runs with the OS user's ambient file authority; the only narrowing in the default configuration is a deny list of home, shallow and credential-named roots. — [src/foundation/workspace.c:279-357](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/foundation/workspace.c#L279-L357); [src/foundation/workspace.c:135-150](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/foundation/workspace.c#L135-L150) (verified)
  - *To reach the next level:* No allowlist by default: indexing is unconfined apart from the always-refused roots unless the operator sets CBM_ALLOWED_ROOT or records grants.
- **C L2:** Both indexing entry points (the MCP tool and the optional UI index route) call the same root policy, and file reads go through a realpath containment check against the indexed root. — [src/mcp/mcp.c:11604-11614](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L11604-L11614); [src/ui/http_server.c:1193](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/ui/http_server.c#L1193); [src/foundation/git_env.h:38-40](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/foundation/git_env.h#L38-L40) (verified)
  - *To reach the next level:* No per-request authorization layer beyond the root policy; helper subprocesses inherit the full user environment apart from git's repository variables.
- **D L2:** Widening needs an operator action (CBM_ALLOWED_ROOT unset is the default; sensitive roots need an explicit allow-root --approve-sensitive from the CLI), and no MCP tool can add grants. — [src/foundation/workspace.c:580-592](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/foundation/workspace.c#L580-L592); [src/main.c:1207-1209](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/main.c#L1207-L1209) (verified)
  - *To reach the next level:* The default is not near-minimal: everything outside the deny list is allowed until the operator declares a boundary.
- **B L3:** A hijacked server can read user-readable source outside the denied roots and write only its own cache databases, ADR text and an opt-in artifact folder in the repository; there are no external credentials or network calls. — [src/mcp/mcp.c:859-863](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L859-L863); searched `rg -n 'curl_easy|getaddrinfo'` in `src` → 0 hits (No outbound HTTP client or DNS resolution in the server source.) (verified)
  - *To reach the next level:* Reads are not confined to one project by default, and the authority is the long-lived OS user rather than a scoped identity.
- **Cap:** none

### C2 Approval gates — 0.62 (high)

As a tool server it relies on the MCP host to ask the user before calls, and it gives the host accurate information to decide with: every one of its 17 tools carries hard-coded read-only and destructive labels, and only index_repository, manage_adr and delete_project are marked as writing. A server-enforced read-only tool profile exists, but it is only used when the client launches the server with that flag; the default entry exposes all tools. There is no dry-run or preview for the destructive tools, and deleting an index or replacing an ADR has no undo, though both affect only local data.

- **S L2:** Separate read and write tools with explicit readOnlyHint/destructiveHint on every tool; an unknown tool name falls back to destructive and not read-only. — [src/mcp/mcp.c:847-865](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L847-L865); [src/mcp/mcp.c:906-912](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L906-L912) (verified)
  - *To reach the next level:* No preview or dry-run for delete_project or manage_adr writes, and the read-only profile is not on by default.
- **C L3:** All 17 tools are listed in the one annotation table, the read-only handlers resolve stores through a query-only path, and the dispatcher rejects any tool outside the active profile. — [src/mcp/mcp.c:833-846](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L833-L846); [src/mcp/mcp.c:18207-18211](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L18207-L18211) (verified)
  - *To reach the next level:* Unknown tools are labelled, not rejected, by the annotation layer, and no server-side confirmation step exists for the three write tools.
- **D L3:** Annotations and profiles are compiled in; the profile is chosen only by a launch flag, and nothing the model, tool input or a workspace file supplies can change them. — [src/mcp/mcp.c:955-962](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L955-L962); [src/mcp/mcp.c:917-933](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L917-L933) (verified)
  - *To reach the next level:* Elevated (full) profile is the default and is not session- or time-bounded.
- **B L2:** A wrongly approved call can delete a project index (rebuildable by reindexing) or overwrite the stored ADR with no undo; no external or high-impact action exists. — [src/mcp/mcp.c:6996-7002](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L6996-L7002); [src/mcp/mcp.c:17866-17868](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L17866-L17868) (verified)
  - *To reach the next level:* No checkpoint or undo for ADR replacement and no preview for deletion.
- **Cap:** none

### C3 Tool & action scoping — 0.57 (high)

Tools are narrow, purpose-built graph operations rather than general shells or HTTP clients. File reads are checked against the indexed root after resolving symlinks, project names are validated, result sizes and traversal depths are clamped, and the Cypher interface accepts only a read subset. Weaker spots: arguments that reach helper commands are checked with a character deny list, the index root is not restricted beyond the sensitive-root deny list by default, and the default tool set includes the three write tools.

- **S L2:** Mix of allowlist-grade checks (realpath containment, project-name validation, numeric clamps) and deny-list filtering of shell metacharacters for helper-command arguments. — [src/mcp/mcp.c:12072-12085](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L12072-L12085); [src/foundation/str_util.c:246-270](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/foundation/str_util.c#L246-L270); [src/mcp/mcp.c:2168-2172](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L2168-L2172) (verified)
  - *To reach the next level:* Helper-command arguments and index roots are checked by deny lists rather than allowlists, and file containment is check-then-open rather than race-safe.
- **C L3:** Every built-in tool resolves projects through the shared validator, file-returning tools share the containment guard, and there are no extension tools. — [src/mcp/mcp.c:12154-12160](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L12154-L12160); [src/mcp/mcp.c:9167-9168](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L9167-L9168) (verified)
  - *To reach the next level:* No single central policy layer that new tools inherit automatically.
- **D L2:** Tool groups are selectable (scout and analysis profiles), but the default profile includes index_repository, manage_adr and delete_project. — [src/mcp/mcp.c:960](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L960); [src/mcp/mcp.c:917-930](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L917-L930) (verified)
  - *To reach the next level:* The default tool set is not read-only.
- **B L2:** Misuse can index and read back source from any non-denied directory and change only local index data; outputs are bounded (snippet 500 lines, trace limit 5000). — [src/mcp/mcp.c:30](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L30); [src/foundation/workspace.c:312-316](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/foundation/workspace.c#L312-L316) (verified)
  - *To reach the next level:* Reads are not scoped to one workspace by default.
- **Cap:** none

### C4 Code-execution isolation — 0.25 (high)

The server has no tool that runs model-written code, but several tools (search_code, detect_changes and the git-history passes) build command lines for git, grep and find as shell strings that include model-supplied values such as search scopes, file patterns and branch names. The only guard is a character filter that rejects quotes, semicolons, pipes and similar metacharacters; there is no sandbox, and the helpers run on the host as the user with the full environment apart from git's repository variables. No second layer sits behind that filter: the helpers have the same reach as the user.

- **S L1:** No isolation primitive: helper commands are run via /bin/sh -c on the host, protected only by metacharacter filtering and quoting. — [src/foundation/compat_fs.c:907-932](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/foundation/compat_fs.c#L907-L932); [src/foundation/str_util.c:246-270](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/foundation/str_util.c#L246-L270) (verified)
  - *To reach the next level:* No argv-based execution or OS sandbox for helper processes.
- **C L1:** The filter is applied on the search_code, detect_changes and git-history paths, but no path is isolated. — [src/mcp/mcp.c:16563](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L16563); [src/mcp/mcp.c:13347](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L13347) (verified)
  - *To reach the next level:* No helper path runs inside any isolation boundary.
- **D L2:** The filtering is always on and cannot be disabled by the model or a workspace file. — [src/mcp/mcp.c:15192-15200](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L15192-L15200) (verified)
  - *To reach the next level:* No sandbox to enable; the protection is filtering only.
- **B L0:** Helpers run as the user on the host with the inherited environment and full network, so a failure of the filter is host-equivalent. — [src/foundation/git_env.h:38-40](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/foundation/git_env.h#L38-L40); [src/foundation/compat_fs.c:932](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/foundation/compat_fs.c#L932) (verified)
  - *To reach the next level:* Helpers would need a minimal environment and no shell interpretation to limit reach.
- **Cap:** none

### C5 Untrusted input blast radius — 0.42 (high)

Everything the server returns comes from repository content the user did not write, such as source code, comments and docstrings. Results are structured: code comes back with its file path and line numbers, and the server's own guidance sits in separate hint fields rather than mixed into the content. Nothing marks repository text as untrusted for the host. The installed hooks also push repository-derived symbol names into some agents' context outside any tool call. The server itself has no outbound channel, but its write tools and the host's own tools remain available to a hijacked model.

- **S L2:** Structured outputs separate returned code from metadata (path, lines) and server hints; server instructions are short workflow guidance without content mixed in. — [src/mcp/mcp.c:1391-1394](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L1391-L1394); [src/mcp/mcp.c:4079](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L4079); searched `rg -n 'untrusted'` in `src/mcp/mcp.c` → 1 hits (The single hit is a comment on path normalization, not an output marker.) (verified)
  - *To reach the next level:* No untrusted/provenance flag the host can act on, and no server mode that drops write tools by default.
- **C L1:** Tool results carry source paths, but hook-injected context and stored ADR text reach the model with no marking. — [src/cli/hook_augment.c:1-12](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/cli/hook_augment.c#L1-L12); [src/cli/cli.c:3397](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/cli/cli.c#L3397) (verified)
  - *To reach the next level:* Repository content on every path, including hook output and ADR text, is not distinguished from trusted text.
- **D L2:** The structured output format is fixed in code and cannot be turned off by content. — [src/mcp/mcp.c:1391-1394](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L1391-L1394) (verified)
  - *To reach the next level:* Nothing beyond output structure is on by default.
- **B L2:** A hijacked host can use this server only to read local source and change local index or ADR data; the server offers no egress, and its write tools are labelled for host approval. — searched `rg -n 'curl_easy|getaddrinfo'` in `src` → 0 hits (No outbound client in the server.); [src/mcp/mcp.c:847-865](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L847-L865) (verified)
  - *To reach the next level:* Sensitive source anywhere on disk outside the denied roots is reachable, and the write tools are exposed in the default profile.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.30 (high)

The server keeps a persistent graph per project and an architecture decision record (ADR) that the model can overwrite at will through manage_adr; that text is stored in the project database and returned in later sessions with no review or history. When a repository contains a committed graph snapshot (.codebase-memory/graph.db.zst) and there is no local index yet, it is imported automatically; the source notes that imported content is trusted as-is. The only repository file that can widen what the server indexes, a manifest of extra roots, needs an explicit approval from the command line that lapses when the file changes. The per-project config file can only map file extensions.

- **S L1:** ADR writes are unvalidated and unversioned, and repository-supplied graph snapshots are imported without a trust decision; only the extra-roots manifest requires a content-keyed human approval. — [src/mcp/mcp.c:10135-10140](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L10135-L10140); [src/mcp/mcp.c:17866-17868](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L17866-L17868); [src/main.c:1207-1209](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/main.c#L1207-L1209) (verified)
  - *To reach the next level:* Memory writes and snapshot imports need validation, review or a trust decision, with history to roll back.
- **C L1:** Only the extra-roots manifest is controlled; ADR text and imported snapshots are not. — [src/mcp/mcp.c:10154-10155](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L10154-L10155); [src/discover/userconfig.c:200](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/discover/userconfig.c#L200) (verified)
  - *To reach the next level:* ADR store and snapshot import are outside any control.
- **D L2:** Each project has its own database file in the per-user cache; there is one local user, but the model can write the ADR of any indexed project. — [src/mcp/mcp.c:2168-2176](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L2168-L2176) (verified)
  - *To reach the next level:* Model writes are not restricted to the current project's namespace.
- **B L1:** Poisoned ADR text or graph content persists across all of the user's sessions and every agent sharing the cache, and when a team commits the opt-in snapshot it travels to teammates; it can steer host models that hold their own tools. — [src/mcp/mcp.c:10141-10150](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L10141-L10150); [src/mcp/mcp.c:11733](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L11733) (verified)
  - *To reach the next level:* No review gate or rollback for persisted ADR text and imported snapshots.
- **Cap:** none

### C7 Third-party extensions — 1.00 (high)

The server loads no third-party code at runtime: its 158 language grammars, the embedding model and SQLite are compiled into the binary, SQLite extension loading is compiled out, and it installs no packages or plugins. Updates are done by the install script or a package manager, never by the running server.

- **Structural absence:** searched `rg -n 'dlopen\(|sqlite3_load_extension|pip install|npm install'` in `src` → 0 hits (No dynamic loading, SQLite extension loading or package installation in the server source.); [Makefile.cbm:583](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/Makefile.cbm#L583)

### C8 Secrets & sensitive-data protection — 0.65 (high)

The server holds no API keys or tokens, sends nothing to any third party and has no telemetry. Its configuration scanners drop values that look like secrets before storing them in the graph, private key files are skipped during discovery, and the request log records only tool names and timings, never arguments or results. Source returned to the model is not redacted, though: search_code and get_code_snippet return whatever the files contain, and helper processes inherit the full environment.

- **S L2:** Secret-pattern filtering in the env and infrastructure scanners and content-free request logging in owner-only log files. — [src/pipeline/pass_infrascan.c:264-271](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/pipeline/pass_infrascan.c#L264-L271); [src/pipeline/pass_envscan.c:493](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/pipeline/pass_envscan.c#L493); [src/foundation/log.c:365-372](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/foundation/log.c#L365-L372) (verified)
  - *To reach the next level:* No redaction of secrets in content returned to the model.
- **C L2:** Logs and the stored graph are covered; model-bound search and snippet output and subprocess environments are not. — [src/mcp/mcp.c:13358](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L13358); [src/discover/discover.c:88](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/discover/discover.c#L88) (verified)
  - *To reach the next level:* Model-bound results and helper environments are not filtered.
- **D L3:** No telemetry or crash reporting exists; diagnostics are opt-in and record only resource counters; secret filtering in scans is always on. — searched `rg -n -i 'sentry|posthog|telemetry'` in `src` → 5 hits (Hits are code comments and an OpenTelemetry trace parser; no telemetry client exists.); [src/foundation/diagnostics.h:4](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/foundation/diagnostics.h#L4) (verified)
  - *To reach the next level:* Stored indexes (which contain source text) are not encrypted or minimised.
- **B L4:** Ambient-only design that accepts no key material, so there is nothing of the server's own to leak; residual exposure is repository content returned to the host model. — searched `rg -n 'curl_easy|getaddrinfo'` in `src` → 0 hits (No outbound client; the server takes no API key.) (verified)
- **Cap:** none

### C9 Audit & traceability — 0.38 (high)

Every tool call handled by the shared daemon is logged with the tool name, error status and duration to an owner-only log file in the user's cache folder, outside the indexed repository. The log does not record arguments, the project acted on or which client asked, so it cannot show what a call actually did. The file is capped at 5 MB with one rotated copy, and logging failures do not stop actions.

- **S L1:** Structured per-call log lines carry the tool name, status and duration but no arguments or target. — [src/foundation/log.c:365-372](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/foundation/log.c#L365-L372); [src/mcp/mcp.c:18664](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L18664) (verified)
  - *To reach the next level:* No arguments, target project or requesting client in the record.
- **C L2:** All tool calls pass through the one dispatcher that logs them; there are no extensions or sub-agents. — [src/mcp/mcp.c:18664](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L18664) (verified)
  - *To reach the next level:* Index writes, ADR changes and deletions are not recorded with their details.
- **D L2:** On by default at the daemon's info level and written to the cache log directory, outside the workspace; the server process could still alter it. — [src/foundation/log.c:109-110](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/foundation/log.c#L109-L110); [src/daemon/host.c:139](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/daemon/host.c#L139) (verified)
  - *To reach the next level:* The record is written by the same process it describes.
- **B L1:** Best-effort logging with a 5 MB cap and a single rotated copy; actions proceed if logging fails. — [src/daemon/host.c:56](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/daemon/host.c#L56) (verified)
  - *To reach the next level:* Records are not durable per action and cannot replay a trajectory.
- **Cap:** none

### C10 Limits & kill switch — 0.50 (high)

The server bounds most of its own work: Cypher queries and code searches stop after 30 seconds, traversal limits and snippet sizes are clamped to hard ceilings, and in-flight indexing and searches can be cancelled when the client goes away. Indexing size limits are off by default, the stdio bridge waits up to 24 hours for a request, and asynchronous index jobs and the background watcher keep running until the shared daemon stops.

- **S L2:** Server-enforced wall-clock deadlines on queries and searches, output caps on traces and snippets, and cancellation of in-flight requests. — [src/cypher/cypher.c:3057](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/cypher/cypher.c#L3057); [src/mcp/mcp.c:145](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L145); [src/mcp/mcp.c:2096-2097](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L2096-L2097) (verified)
  - *To reach the next level:* Not every operation is capped (index size limits default to off) and there are no rate limits on tool calls.
- **C L2:** Deadlines cover the query and search paths and their helper processes; background watcher re-indexing and async index jobs run outside those per-call limits. — [src/cli/cli.c:7575](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/cli/cli.c#L7575); [src/daemon/frontend.c:23](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/daemon/frontend.c#L23) (verified)
  - *To reach the next level:* Background and async work does not count against a shared budget.
- **D L2:** Sensible defaults with hard ceilings on caller-raised limits (trace limit at most 5000); index size caps are operator settings that default to off. — [src/cli/cli.c:7583-7584](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/cli/cli.c#L7583-L7584); [src/mcp/mcp.c:9167-9168](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/mcp/mcp.c#L9167-L9168) (verified)
  - *To reach the next level:* Index size limits are off by default.
- **B L2:** A runaway query or search is cut off at 30 seconds, but an index of a very large tree is bounded only by memory budget and progress timeouts, and async jobs continue after the caller disconnects until the daemon exits. — [src/daemon/frontend.c:23](https://github.com/DeusData/codebase-memory-mcp/blob/268a9d8886642eb7f9b2ce45f5ce27cdecf0f519/src/daemon/frontend.c#L23) (verified)
  - *To reach the next level:* Async jobs and the watcher outlive the requesting call.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Repository source returned by get_code_snippet/search_code (src/mcp/mcp.c:12158) · [B] sensitive data/systems: Any user-readable directory outside the denied roots can be indexed and read (src/foundation/workspace.c:332) · [C] state change / egress: Local writes only: manage_adr, delete_project, index persistence (src/mcp/mcp.c:859); no egress · Same default session? Yes

## Highest-impact improvements
1. Run git, grep and find helpers through the existing argv-based subprocess API with a minimal environment instead of /bin/sh command strings. (C4 B L0→L2, +0.100 before caps; Playbook 3)
2. Require a content-keyed trust approval (as already done for the roots manifest) before importing a repository-supplied graph snapshot, and keep ADR history so changes can be reviewed and rolled back. (C6 C L1→L2, +0.075 before caps; Playbook 2)
3. Log each tool call's arguments (or a redacted digest), target project and requesting client alongside the existing name and status. (C9 S L1→L2, +0.075 before caps; Playbook 1 step 3)
4. Add a preview or dry-run mode for delete_project and ADR replacement. (C2 S L2→L3, +0.075 before caps; Playbook 5)
5. Make the read-only analysis profile the default for the main MCP entry and expose write tools only on explicit opt-in. (C3 D L2→L3, +0.050 before caps; Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The installer (45 client integrations in src/cli) was reviewed only for auto-approve or permission grants and the hook design; per-client config writers were not audited line by line.
- The opt-in graph UI HTTP server was reviewed only for its default (off), loopback binding and origin checks; Windows-specific command paths, the vendored tree-sitter grammars, SQLite and the LSP resolution passes were not examined.
- No attempts to steer AI reviewers were found in the files read.
