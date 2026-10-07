# Defense-in-Depth Score: IDA Pro MCP

**Repo:** https://github.com/mrexodia/ida-pro-mcp · **Commit:** `c133c3853faa111a9b00ee615c013b720d0c4acd` · **Reviewed:** 2026-10-03
**What it is:** MCP server bridging IDA Pro with LLMs for reverse engineering (incl. debugger ops)
**Category:** Cybersecurity
**Scored configuration:** Headless idalib-mcp supervisor over stdio exactly as the shipped Claude Code plugin launches it (no --unsafe, no --profile); the legacy IDA GUI plugin, which enables every tool by default, is footnoted.
**Agent surface (default):** code execution opt-in · filesystem write yes · network egress no · external credentials no · persistent memory yes · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 3.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L1 | L2 | 0.15 | — | **0.15** | High |
| C2 | Approval gates | L2 | L2 | L0 | L2 | 0.40 | G1 | **0.40** (alt) | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C4 | Code-execution isolation | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C5 | Untrusted input blast radius | L2 | L0 | L0 | L1 | 0.20 | G1 | **0.20** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L1 | L1 | 0.25 | C6-REPOCONFIG | **0.25** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L3 | 0.40 | — | **0.40** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |

Controls where a risk surface exists: 2.80 / 9.0 (31%); 1 criterion scored SA (surface absent).

As the Claude Code plugin launches it, IDA Pro MCP is a local stdio server with no authentication and no per-call gating: the code-execution and debugger tools are withheld, but the database-editing and file-saving tools are on, and nothing separates attacker-controlled binary strings from instructions. Enforcement of that tier does not cover every path, and workers keep running after the supervisor exits. A read-only profile exists but is opt-in; the legacy GUI plugin enables every tool, including in-process Python, by default.

## Critical gaps
- Python execution tools run model code in-process with no isolation, and spawned workers inherit the full environment. (ASI05; C4) — [src/ida_pro_mcp/ida_mcp/api_python.py:189](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_python.py#L189); [src/ida_pro_mcp/idalib_supervisor.py:279](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_supervisor.py#L279)

## Criterion details

### C1 Identity & least privilege — 0.15 (high)

The server holds no credentials of its own and never authenticates a caller: the stdio supervisor and the loopback HTTP workers it spawns accept any request, and the workers keep running after the supervisor exits. Nothing narrows the operating-system user's authority; idb_open will open any path the user can read and idb_save will write a database copy to any path they can write. The only default narrowing is that the code-execution and debugger tools are withheld unless an operator passes a command-line flag. A hijacked agent therefore acts with the full file authority of the user, limited to what the exposed tools can do.

- **S L0:** No authentication or authorization layer exists; the server runs with the launching user's ambient authority and idb_open/idb_save take arbitrary paths. — [src/ida_pro_mcp/idalib_supervisor.py:1289](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_supervisor.py#L1289); [src/ida_pro_mcp/ida_mcp/api_core.py:861](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_core.py#L861); searched `rg -n -i 'bearer|api_key|apikey|auth_token|Authorization' src/ida_pro_mcp --glob '!**/tests/**'` in `src/ida_pro_mcp` → 0 hits (no authentication primitive of any kind; the few hits (if any) are unrelated words) (verified)
  - *To reach the next level:* L1 needs a dedicated or scoped identity; there is no token, no per-request check and no path restriction.
- **C L0:** Authorization is applied to no tool path; the supervisor forwards every tools/call to a worker without any caller check. — [src/ida_pro_mcp/idalib_supervisor.py:1383](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_supervisor.py#L1383); [src/ida_pro_mcp/ida_mcp/zeromcp/mcp.py:303](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/zeromcp/mcp.py#L303) (verified)
  - *To reach the next level:* L1 needs at least the main tool path to be authorization-checked.
- **D L1:** Narrower default than the full tool set: unsafe tools are removed from the headless worker unless --unsafe is passed, but that narrowing is not enforced on every path and the GUI plugin enables everything by default. — [src/ida_pro_mcp/idalib_server.py:289](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_server.py#L289) (verified)
  - *To reach the next level:* L2 needs the narrower default to be changeable only by an operator action.
- **B L2:** A hijacked caller can modify IDA databases and write database copies to arbitrary paths, and open any readable file for analysis, but the server holds no credentials for other systems. — [src/ida_pro_mcp/ida_mcp/api_core.py:881](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_core.py#L881); [src/ida_pro_mcp/ida_mcp/api_memory.py:266](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_memory.py#L266) (verified)
  - *To reach the next level:* L3 needs scoping to one project with non-destructive writes only; idb_save to any path and patch tools are destructive.
- **Cap:** none

### C2 Approval gates — 0.40 (high)

As a tool server the project leaves approval to the host, so what matters is the risk signalling it gives the host. It publishes no readOnlyHint or destructiveHint annotations, so a host cannot tell the byte-patching, renaming and save tools from the read-only ones. The one server-enforced tier is the unsafe set (Python execution and the debugger), which is withheld by default in the headless server; however enforcement of that tier does not cover every path. Mutating tools such as patch, patch_asm, put_int and idb_save are not in the unsafe set, and only rename offers a dry run.

- **default configuration** (default; raw 0.30 → 0.30)
  - **S L1:** Read and write tools are separate and a server-enforced unsafe tier exists, but no tool carries risk annotations and only rename has a dry run. — searched `rg -n 'readOnlyHint|destructiveHint|idempotentHint' src/ida_pro_mcp` in `src/ida_pro_mcp` → 0 hits (no annotation is emitted anywhere in tools/list generation); [src/ida_pro_mcp/ida_mcp/zeromcp/mcp.py:1380](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/zeromcp/mcp.py#L1380); [src/ida_pro_mcp/ida_mcp/api_modify.py:412](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_modify.py#L412) (verified)
    - *To reach the next level:* L2 needs accurate readOnlyHint/destructiveHint annotations on every tool.
  - **C L1:** Only the tools tagged @unsafe are withheld; other mutating tools (patch, put_int, patch_asm, idb_save) are neither flagged nor withheld. — [src/ida_pro_mcp/ida_mcp/rpc.py:201](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/rpc.py#L201); [src/ida_pro_mcp/ida_mcp/api_memory.py:266](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_memory.py#L266); [src/ida_pro_mcp/ida_mcp/api_modify.py:326](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_modify.py#L326) (verified)
    - *To reach the next level:* L2 needs every mutating built-in tool to be flagged or gated.
  - **D L1:** Unsafe tools are withheld by default in the headless server, but that enforcement does not cover every path. (verified)
    - *To reach the next level:* L2 needs the tier to be changeable only by the operator launching the server.
  - **B L2:** Wrongly approved edits change an IDA database (IDA keeps original bytes, and rename has dry_run) but there is no checkpoint or rollback, and idb_save can overwrite a file at an arbitrary path. — [src/ida_pro_mcp/ida_mcp/api_modify.py:412](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_modify.py#L412); [src/ida_pro_mcp/ida_mcp/api_core.py:861](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_core.py#L861) (verified)
    - *To reach the next level:* L3 needs rollback or previews for destructive operations.
- **opt-in read-only profile (profiles/readonly.txt via --profile)** (alt; raw 0.40, cap G1 → 0.40) ← counted
  - **S L2:** A shipped allowlist exposes only analysis tools that never mutate the database and is enforced server-side by removing every other tool, but tools still carry no annotations. — [profiles/readonly.txt:1](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/profiles/readonly.txt#L1); [src/ida_pro_mcp/ida_mcp/profile.py:36](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/profile.py#L36) (verified)
    - *To reach the next level:* L3 needs annotations on every tool plus dry-runs for destructive operations.
  - **C L2:** Every tool not listed is removed from the worker's tool table, but the profile is not enforced on every path. — [src/ida_pro_mcp/ida_mcp/profile.py:36](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/profile.py#L36) (verified)
    - *To reach the next level:* L3 needs every path to respect the profile.
  - **D L0:** Opt-in only: the flag must be passed and the shipped plugin does not pass it. — [.claude-plugin/plugin.json:12](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/.claude-plugin/plugin.json#L12) (verified)
    - *To reach the next level:* Needs to be on by default.
  - **B L2:** With the profile only idb_open, idb_list and analysis reads remain; opening a path still loads an arbitrary file into IDA and writes database files. — [profiles/readonly.txt:1](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/profiles/readonly.txt#L1); [src/ida_pro_mcp/idalib_server.py:61](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_server.py#L61) (verified)
    - *To reach the next level:* L3 needs previews for the remaining state-changing management tools.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.
- **Notes:** The GUI plugin path (ida_mcp.py) autostarts and enables every tool including py_eval and the debugger by default; the README says it is no longer recommended.

### C3 Tool & action scoping — 0.50 (high)

Tools have typed schemas generated from Python type hints and addresses are parsed through a helper, but there is no allowlist validation of paths: idb_open accepts any path, idb_save writes to any path, and py_exec_file (unsafe, withheld by default) executes any file. In the default headless configuration the code-execution and debugger tools are removed, which leaves analysis and database-editing tools; the editing tools (patch, rename, set_type, patch_asm) are on by default. Everything is scoped to the open IDA databases, with full write access inside them.

- **S L2:** Typed schemas from type hints plus address parsing, but paths are unvalidated strings. — [src/ida_pro_mcp/ida_mcp/zeromcp/jsonrpc.py:254](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/zeromcp/jsonrpc.py#L254); [src/ida_pro_mcp/ida_mcp/utils.py:608](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/utils.py#L608); [src/ida_pro_mcp/idalib_server.py:104](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_server.py#L104) (verified)
  - *To reach the next level:* L3 needs resolved-path containment for idb_open/idb_save and bounds on quantities.
- **C L2:** Schema typing and address parsing apply to every worker tool; path-taking tools have no validation. — [src/ida_pro_mcp/ida_mcp/zeromcp/jsonrpc.py:254](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/zeromcp/jsonrpc.py#L254); [src/ida_pro_mcp/ida_mcp/api_core.py:862](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_core.py#L862) (verified)
  - *To reach the next level:* L3 needs a shared validation layer covering all built-in tools including path arguments.
- **D L2:** Default headless set removes exec and debugger tools but keeps write tools; groups can be narrowed with --profile. — [src/ida_pro_mcp/idalib_server.py:289](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_server.py#L289); [src/ida_pro_mcp/ida_mcp/api_memory.py:266](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_memory.py#L266) (verified)
  - *To reach the next level:* L3 needs a read-only default with writes enabled only on request.
- **B L2:** Scoped to open IDA databases with full write inside them; paths reach any file the user can read or write. — [src/ida_pro_mcp/ida_mcp/api_core.py:881](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_core.py#L881); [src/ida_pro_mcp/ida_mcp/api_python.py:225](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_python.py#L225) (verified)
  - *To reach the next level:* L3 needs quantity bounds and path scoping.
- **Cap:** none

### C4 Code-execution isolation — 0.05 (high)

There is no containment for code execution. When the Python tools are enabled, model-supplied code runs with exec and eval inside the IDA process with the full builtins and the process's complete environment and file access; the debugger tools launch the analyzed binary under IDA. By default the headless server withholds those tools, which is a real reduction in exposure but not isolation, and that enforcement does not cover every path. The spawned worker processes inherit the whole parent environment, so any code that does run can read the user's secrets.

- **S L0:** py_eval and py_exec_file run model-supplied code via exec/eval in-process with unrestricted builtins; the debugger tools start the analyzed binary. — [src/ida_pro_mcp/ida_mcp/api_python.py:189](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_python.py#L189); [src/ida_pro_mcp/ida_mcp/api_python.py:254](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_python.py#L254); [src/ida_pro_mcp/ida_mcp/api_debug.py:411](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_debug.py#L411) (verified)
  - *To reach the next level:* L1 would need at least filtering; none exists.
- **C L0:** Every execution path (py_eval, py_exec_file, dbg_start) runs directly in the host process; none is sandboxed. — [src/ida_pro_mcp/ida_mcp/api_python.py:144](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_python.py#L144); [src/ida_pro_mcp/ida_mcp/api_python.py:246](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_python.py#L246) (verified)
  - *To reach the next level:* L1 needs the main exec path sandboxed.
- **D L1:** Exec tools are withheld by default in the headless server and need an operator flag, but there is no sandbox; that enforcement does not cover every path, and the GUI plugin exposes them by default. — [src/ida_pro_mcp/idalib_server.py:289](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_server.py#L289) (verified)
  - *To reach the next level:* L2 needs an actual boundary that is on by default.
- **B L0:** A worker is spawned without an env argument so it inherits the parent's full environment, and exec'd code runs in-process with that environment and the user's files and network. — [src/ida_pro_mcp/idalib_supervisor.py:279](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_supervisor.py#L279); searched `rg -n 'env=' src/ida_pro_mcp/idalib_supervisor.py` in `src/ida_pro_mcp/idalib_supervisor.py` → 0 hits (no env= argument is passed to the Popen call, so the full environment is inherited; hits would be unrelated) (verified)
  - *To reach the next level:* L1 needs the process to hold no credentials in its environment.
- **Cap:** none

### C5 Untrusted input blast radius — 0.20 (high)

Everything the server returns is derived from the binary under analysis, which is attacker-controlled in malware work: strings, symbol names, comments and decompiled text. Outputs are structured JSON with schemas, but nothing marks content as untrusted or records its source, and the server offers no mode that cuts off state changes after untrusted content has been read, other than an opt-in read-only profile. The server has no network tool and holds no credentials, so exfiltration depends on the host; irreversible effects are limited to overwriting a file via idb_save and editing the database.

- **S L2:** Tool outputs are structured (structuredContent with output schemas) separating data from metadata, with no provenance or untrusted flag. — [src/ida_pro_mcp/ida_mcp/zeromcp/mcp.py:1075](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/zeromcp/mcp.py#L1075); searched `rg -n -i 'untrusted|provenance|tainted' src/ida_pro_mcp --glob '!**/tests/**'` in `src/ida_pro_mcp` → 0 hits (no provenance or untrusted marking exists; the hits (if any) are unrelated) (verified)
  - *To reach the next level:* L3 needs source/untrusted provenance on returned content.
- **C L0:** Binary-derived strings, names and comments enter context with the same standing as any tool output; no source is distinguished. — [src/ida_pro_mcp/ida_mcp/zeromcp/mcp.py:1075](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/zeromcp/mcp.py#L1075) (verified)
  - *To reach the next level:* L1 needs at least one source handled.
- **D L0:** The only mechanism that drops a Rule-of-Two leg (the read-only profile) is opt-in and not used by the shipped plugin. — [.claude-plugin/plugin.json:12](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/.claude-plugin/plugin.json#L12); [profiles/readonly.txt:1](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/profiles/readonly.txt#L1) (verified)
  - *To reach the next level:* Needs a protective mode on by default.
- **B L1:** After a hijack the attacker can run the unflagged write tools and overwrite a file via idb_save with no server-side gate; the server has no egress channel and no secrets of its own, so leak-plus-irreversible depends on host tools. — [src/ida_pro_mcp/ida_mcp/api_core.py:881](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_core.py#L881); [src/ida_pro_mcp/ida_mcp/api_memory.py:266](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_memory.py#L266) (verified)
  - *To reach the next level:* L2 needs destructive actions to require server-side confirmation.
- **Cap:** G1 — The only control that drops a Rule-of-Two leg (the read-only profile) is opt-in and not used by the shipped plugin.
- **Notes:** A DEFAULT_ANALYSIS_PROMPT system_reminder string and get_analysis_prompt exist in utils.py but have no callers at this commit; dbg tool docstrings and errors carry server-authored instructions to the model.

### C6 Memory, context & configuration integrity — 0.25 (high)

Model edits persist: names, comments, types and patches are saved into the IDA database and re-appear in later sessions' tool output with no provenance, and every call is logged into the same database. The server also keeps some of its own configuration inside the IDA database, which is the untrusted artifact under analysis, and that configuration is not integrity-protected.

- **S L1:** Writes are logged by the always-on tracer but not validated, and re-surface in outputs without provenance; database-resident configuration is not integrity-protected. — [src/ida_pro_mcp/ida_mcp/trace.py:296](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/trace.py#L296) (verified)
  - *To reach the next level:* L2 needs provenance on persisted content and controls on database-resident configuration.
- **C L1:** Only the call trace covers writes; persisted comments are not controlled and database-resident settings are not integrity-protected. (verified)
  - *To reach the next level:* L2 needs the main store and the config store controlled.
- **D L1:** Isolation is per IDA database via the session routing, not per user; any tool call can open or write another path. — [src/ida_pro_mcp/idalib_supervisor.py:1383](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_supervisor.py#L1383); [src/ida_pro_mcp/idalib_supervisor.py:1289](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_supervisor.py#L1289) (verified)
  - *To reach the next level:* L2 needs namespaces enforced in queries.
- **B L1:** Poisoned comments or names persist in the saved database across sessions and are fed back to the model, which can then drive tools. — [src/ida_pro_mcp/ida_mcp/api_modify.py:164](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_modify.py#L164); [src/ida_pro_mcp/ida_mcp/api_core.py:861](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/api_core.py#L861) (verified)
  - *To reach the next level:* L2 needs persisted content to influence text only or gated actions.
- **Cap:** C6-REPOCONFIG — Security-relevant configuration is influenced by the opened IDA database.
- **Notes:** The score is already below the cap; the cap is recorded because the trigger is verified in source.

### C7 Third-party extensions — 1.00 (high)

The server loads no third-party plugins, models, packages or remote tools at runtime; the supervisor launches only its own worker module and, for GUI sessions, can adopt instances listed in files under the user's home directory (a local, same-user mechanism noted below). The absence is by design rather than a control, so the criterion is marked as structurally absent.

- **Structural absence:** searched `rg -n -i 'entry_points|pkg_resources|pip install|trust_remote_code|pickle|torch\.load|load_plugin|marshal' src/ida_pro_mcp --glob '!**/tests/**'` in `src/ida_pro_mcp` → 0 hits (no dynamic extension, deserialization or install path); searched `rg -n 'importlib' src/ida_pro_mcp --glob '!**/tests/**'` in `src/ida_pro_mcp` → 12 hits (hits load the project's own discovery module by path and its own test modules, not third-party code)
- **Notes:** The supervisor adopts GUI/worker instances listed in files under ~/.idapro/mcp/instances and forwards calls to the host and port recorded there (discovery.py, idalib_supervisor.py _make_gui_session) after only a PID and TCP probe; that is a user-scope local registry, not downloaded third-party code, but any same-user process can add an entry.

### C8 Secrets & sensitive-data protection — 0.40 (high)

The server accepts no API keys or tokens and has no telemetry or crash reporting, so there is little secret material of its own to leak. It does not scrub anything either: spawned workers inherit the parent's whole environment, request parameters are logged truncated at debug level, and every tool call with its full arguments and result is stored in the IDA database. The model can also read any file the user can open by loading it with idb_open.

- **S L1:** No credential handling exists and nothing is masked or scrubbed; the server relies on holding no keys. — searched `rg -n -i 'api_key|apikey|password|secret|credential' src/ida_pro_mcp --glob '!**/tests/**'` in `src/ida_pro_mcp` → 0 hits (no secret handling code; hits (if any) are unrelated words); [src/ida_pro_mcp/ida_mcp/zeromcp/jsonrpc.py:141](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/zeromcp/jsonrpc.py#L141) (verified)
  - *To reach the next level:* L2 needs redaction or masking on the main paths.
- **C L1:** Only one path has any limiting: debug request logs truncate parameters to 200 characters; trace records and subprocess environments are unfiltered. — [src/ida_pro_mcp/ida_mcp/zeromcp/jsonrpc.py:142](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/zeromcp/jsonrpc.py#L142); [src/ida_pro_mcp/ida_mcp/trace.py:313](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/trace.py#L313) (verified)
  - *To reach the next level:* L2 needs logs and transcripts handled.
- **D L2:** No telemetry or crash reporter exists; logging defaults are local and moderate, with no redaction to disable. — searched `rg -n -i 'sentry|posthog|telemetry|analytics' src/ida_pro_mcp --glob '!**/tests/**'` in `src/ida_pro_mcp` → 0 hits (no telemetry code) (verified)
  - *To reach the next level:* L3 needs redaction always on.
- **B L3:** The server holds no keys; the exposure is the inherited environment of workers and the user's readable files, reachable only through enabled tools. — [src/ida_pro_mcp/idalib_supervisor.py:279](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_supervisor.py#L279); [src/ida_pro_mcp/idalib_supervisor.py:1289](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_supervisor.py#L1289) (verified)
  - *To reach the next level:* L4 needs worker environments scrubbed so residual gaps are closed.
- **Cap:** none

### C9 Audit & traceability — 0.45 (high)

Every tools/call handled by a worker is recorded with tool name, arguments, structured result, duration and timestamp into an append-only gzipped log inside the IDA database, and this tracer is always on. It does not record who asked or who approved, the supervisor's own idb_open and idb_close calls are not recorded, and records are buffered in batches, with write errors swallowed silently, so a crash can lose recent entries. The log lives in the database file the same process edits.

- **S L2:** Structured record per call with arguments, result, timestamp and duration; no actor, principal or approver fields. — [src/ida_pro_mcp/ida_mcp/trace.py:296](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/trace.py#L296); [src/ida_pro_mcp/ida_mcp/trace.py:301](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/trace.py#L301) (verified)
  - *To reach the next level:* L3 needs actor attribution.
- **C L2:** All worker tools are traced; supervisor-level idb_open/idb_close are not, and approvals do not exist to record. — [src/ida_pro_mcp/ida_mcp/trace.py:325](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/trace.py#L325); searched `rg -n 'trace' src/ida_pro_mcp/idalib_supervisor.py` in `src/ida_pro_mcp/idalib_supervisor.py` → 0 hits (no tracer in the supervisor; no hits) (verified)
  - *To reach the next level:* L3 needs every tool call and approvals/denials recorded.
- **D L2:** Always on with no off switch, stored in the database next to the binary and written by the same process that serves the model; not separated from the agent's process. — [src/ida_pro_mcp/ida_mcp/__init__.py:49](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/__init__.py#L49); [src/ida_pro_mcp/ida_mcp/trace.py:26](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/trace.py#L26) (verified)
  - *To reach the next level:* L3 needs a writer the model cannot influence.
- **B L1:** Writes are buffered (256 records or 64 KiB) and failures are swallowed so actions proceed unlogged; the log is lost if the database is never saved. — [src/ida_pro_mcp/ida_mcp/trace.py:38](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/trace.py#L38); [src/ida_pro_mcp/ida_mcp/trace.py:285](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/trace.py#L285) (verified)
  - *To reach the next level:* L2 needs errors surfaced and records flushed per action.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

The server bounds its own work: tool calls on the worker have a 60 second default timeout with native cancellation, outputs over 50,000 characters are truncated into a cached download, request bodies are capped at 10 MB, and the supervisor limits simultaneous databases to four and gives each forwarded call 900 seconds. There are no rate limits, and workers are deliberately detached so they keep running with full tool authority after the supervisor stops, until an idle timeout the caller can lengthen.

- **S L2:** Server-enforced timeout (default 60s, env-adjustable), output size cap, body cap and worker count cap, with in-flight cancellation, but no rate limits and no proof every operation is covered. — [src/ida_pro_mcp/ida_mcp/sync.py:43](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/sync.py#L43); [src/ida_pro_mcp/ida_mcp/rpc.py:20](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/ida_mcp/rpc.py#L20); [src/ida_pro_mcp/idalib_supervisor.py:1492](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_supervisor.py#L1492) (verified)
  - *To reach the next level:* L3 needs caps on every operation plus rate limits.
- **C L2:** The timeout applies to every worker tool call and the supervisor bounds forwarded calls; spawned workers are capped in number but outlive the supervisor. — [src/ida_pro_mcp/idalib_supervisor.py:85](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_supervisor.py#L85); [src/ida_pro_mcp/idalib_supervisor.py:279](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_supervisor.py#L279) (verified)
  - *To reach the next level:* L3 needs spawned processes to count against the same budget through shutdown.
- **D L1:** Sensible defaults exist, but idb_open lets the caller set idle_ttl_sec, lengthening how long a worker with full tool authority stays alive. — [src/ida_pro_mcp/worker_lifecycle.py:74](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/worker_lifecycle.py#L74); [src/ida_pro_mcp/idalib_supervisor.py:1301](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_supervisor.py#L1301) (verified)
  - *To reach the next level:* L2 needs limits the model cannot raise.
- **B L1:** Stopping the supervisor deliberately leaves detached workers running until idle timeout, each exposing unauthenticated tools on loopback. — [src/ida_pro_mcp/idalib_supervisor.py:376](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_supervisor.py#L376); [src/ida_pro_mcp/idalib_supervisor.py:377](https://github.com/mrexodia/ida-pro-mcp/blob/c133c3853faa111a9b00ee615c013b720d0c4acd/src/ida_pro_mcp/idalib_supervisor.py#L377) (verified)
  - *To reach the next level:* L2 needs stop to end the loop and all spawned processes.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: binary strings, names, comments and decompiled text returned by tools (src/ida_pro_mcp/ida_mcp/api_analysis.py) · [B] sensitive data/systems: any file the user can open via idb_open (src/ida_pro_mcp/idalib_supervisor.py:1289); no credentials held · [C] state change / egress: patch, put_int, patch_asm, rename and idb_save with no gate (src/ida_pro_mcp/ida_mcp/api_memory.py:266); no network egress tool · Same default session? Yes

## Highest-impact improvements
1. Ensure withheld and profile-filtered tools can only be re-enabled by the operator launching the server. — C2 D L1→L3, +0.100 before caps (Playbook 5, step 1)
2. Require a per-launch bearer token on the supervisor and worker HTTP endpoints and reject requests without it. — C1 S L0→L2, +0.150 before caps
3. Emit readOnlyHint/destructiveHint annotations for every tool and mark patch, put_int, patch_asm and idb_save as destructive. — C2 S L1→L2, +0.075 before caps (Playbook 5, step 1)
4. Spawn workers with an explicit minimal environment instead of inheriting the parent's. — C4 B L0→L1, +0.050 before caps (Playbook 3, step 1)
5. Restrict idb_open and idb_save to an operator-configured set of directories after path resolution. — C3 S L2→L3, +0.075 before caps (Playbook 3, step 2)

## Re-audit log
- No changes.

## Limitations
- Static source review of commit c133c38 only; nothing was executed, installed, built or probed.
- Scored the headless idalib-mcp stdio configuration that the Claude Code plugin ships; the legacy IDA GUI plugin (autostart, all tools enabled including py_eval and debugger) was reviewed but is footnoted, not scored. The README states the GUI plugin is no longer recommended and recommends the official Hex-Rays IDA MCP server instead.
- Reachability by the model of the gaps in unsafe-tier enforcement is inferred; no G2 cap was applied.
- IDA Pro itself (loaders, plugins, idalib) and what opening an untrusted binary or database does inside IDA were not examined.
- Model behaviour is out of scope; only code-level controls were scored.
- No reviewer-directed prompt injection found in the repo.
