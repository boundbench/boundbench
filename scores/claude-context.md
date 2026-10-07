# Defense-in-Depth Score: Claude Context

**Repo:** https://github.com/zilliztech/claude-context · **Commit:** `6fc318b4e3ce58e2898b00a9c3538ead9e24dee5` · **Reviewed:** 2026-10-05
**What it is:** MCP server and core library that index codebases into a Milvus/Zilliz vector database and expose semantic code search to coding agents.
**Category:** Coding
**Scored configuration:** Local stdio MCP server (packages/mcp) launched via npx @zilliz/claude-context-mcp@latest with OPENAI_API_KEY, MILVUS_ADDRESS and MILVUS_TOKEN (Zilliz Cloud) as in the README, background sync and trigger watcher on by default.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 4.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L0 | L1 | L2 | 0.23 | — | **0.23** | High |
| C2 | Approval gates | L0 | L0 | L0 | L2 | 0.10 | — | **0.10** | High |
| C3 | Tool & action scoping | L2 | L2 | L0 | L1 | 0.35 | — | **0.35** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L2 | 0.10 | — | **0.10** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C9 | Audit & traceability | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |

Controls where a risk surface exists: 2.10 / 8.0 (26%); 2 criteria scored SA (surface absent).

A narrow code-search server: it runs no code and loads no plugins, and it keeps dotfiles and .env files out of its index. Around that core it has few safeguards. Any directory the user can read can be indexed and uploaded to the embedding provider and vector database, the destructive clear tool carries no risk labels for hosts, and search results return indexed code (including third-party code) unmarked. The index persists and re-syncs in the background, with no audit log of the server's own.

## Critical gaps
- None: no criterion is capped and no blast radius is at the worst level.

## Criterion details

### C1 Identity & least privilege — 0.23 (high)

The server runs as the local OS user and holds two long-lived service keys: an embedding-provider key (OpenAI by default) and a vector-database key, which the README tells users to fill with their Zilliz Cloud Personal Key. Every tool uses the same shared clients, and there is no authorization step: any directory the OS user can read can be indexed, and any indexed codebase can be searched or cleared. Nothing narrows the keys per tool or per request.

- **S L1:** Dedicated service keys for the embedding provider and the vector database are read from the environment, but they are broad (the README points users to an account-level Zilliz Personal Key) and filesystem access is the full OS user. Evidence: [packages/mcp/src/config.ts:173-186](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/config.ts#L173-L186); [README.md:44](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/README.md#L44) (verified)
  - *To reach the next level:* No per-tool or role-scoped credentials and no restriction of which directories the server may read.
- **C L0:** No tool passes an authorization check; all four tools share one embedding client and one vector-database client created at startup. Evidence: [packages/mcp/src/index.ts:62-72](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/index.ts#L62-L72); searched `rg -n -i 'realpath|allowedRoots|allowlist|ALLOWED_'` in `packages/mcp/src packages/core/src` → 0 hits (No root allowlist or path authorization exists.) (verified)
  - *To reach the next level:* An authorization layer every tool passes through (for example a configured set of allowed roots) is missing.
- **D L1:** The documented default configuration uses an account-level vector-database key and the OS user's full read access; narrowing requires the operator to choose a cluster-scoped token on their own. Evidence: [README.md:68-73](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/README.md#L68-L73) (verified)
  - *To reach the next level:* A least-privilege default (cluster-scoped token, configured allowed roots) is not provided.
- **B L2:** A hijacked caller can read any code or markdown file the OS user can read into the index and write to or drop collections in one vector-database account; it cannot run commands or write local files. Evidence: [packages/mcp/src/handlers.ts:352-355](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/handlers.ts#L352-L355); [packages/mcp/src/handlers.ts:928](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/handlers.ts#L928) (verified)
  - *To reach the next level:* Authority is not limited to one project or to read-only operations.
- **Cap:** none

### C2 Approval gates — 0.10 (high)

The server gives the host nothing to base an approval decision on. None of the four tools carries MCP risk annotations, so a host cannot tell the read-only search and status tools from clear_index, which drops a codebase's collection, or index_codebase, which with force=true deletes and rebuilds an index and uploads file contents to the embedding provider. The only safeguard is tool-description text asking the model to confirm with the user before a forced re-index. The destructive actions affect derived index data that can be rebuilt by re-indexing.

- **S L0:** No tool declares readOnlyHint or destructiveHint, and index_codebase mixes creating an index with deleting the existing one (force). Evidence: searched `rg -n -i 'readOnlyHint|destructiveHint|annotations'` in `packages/mcp/src packages/core/src` → 0 hits (No MCP tool annotations anywhere.); [packages/mcp/src/index.ts:95](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/index.ts#L95) (verified)
  - *To reach the next level:* Accurate read/destructive annotations on every tool are missing.
- **C L0:** No tool, including clear_index, carries any risk signal or server-side confirmation step. Evidence: [packages/mcp/src/index.ts:198-199](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/index.ts#L198-L199) (verified)
  - *To reach the next level:* Risk signalling on the mutating tools (clear_index, index_codebase) is missing.
- **D L0:** There is no read-only mode or confirmation setting to enable; every tool is exposed as-is. Evidence: [packages/mcp/src/index.ts:233-245](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/index.ts#L233-L245) (verified)
  - *To reach the next level:* A server-enforced read-only mode or confirmation step is missing.
- **B L2:** A wrongly approved call can drop or rebuild an index, which is derived data recoverable by re-indexing, but uploads of file contents to the embedding provider cannot be undone. Evidence: [packages/mcp/src/handlers.ts:429-435](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/handlers.ts#L429-L435) (verified)
  - *To reach the next level:* No preview or dry-run for destructive or uploading operations.
- **Cap:** none

### C3 Tool & action scoping — 0.35 (high)

The tools are narrow (index, search, clear, status) and there is no shell, generic HTTP or file-write tool. Validation is light: paths are made absolute and checked to be existing directories, the splitter is checked against an enum, search results are capped at 50 and snippets at 5,000 characters, and indexing is limited by a default extension allowlist that skips dotfiles and .env files. There is no containment to a project root, so any readable directory can be indexed, and the model-supplied extension and ignore-pattern lists are taken without checks, which lets a caller widen what gets indexed.

- **S L2:** Typed schemas plus some checks (existing directory, splitter enum, result cap), but no resolved-path containment and no validation of custom extension or ignore-pattern lists. Evidence: [packages/mcp/src/utils.ts:16-25](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/utils.ts#L16-L25); [packages/mcp/src/handlers.ts:336](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/handlers.ts#L336); [packages/mcp/src/handlers.ts:765](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/handlers.ts#L765) (verified)
  - *To reach the next level:* Allowlist validation such as containment to configured roots is missing.
- **C L2:** All four tools apply the same path checks, but the indexing options (customExtensions, ignorePatterns) are passed through unvalidated. Evidence: [packages/mcp/src/handlers.ts:325-329](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/handlers.ts#L325-L329) (verified)
  - *To reach the next level:* Not every argument of every tool is validated.
- **D L0:** The full tool set, including the destructive clear_index and the uploading index_codebase, is always enabled and cannot be reduced by configuration. Evidence: [packages/mcp/src/index.ts:122-125](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/index.ts#L122-L125) (verified)
  - *To reach the next level:* Tools cannot be disabled individually and there is no read-only default.
- **B L1:** A misused tool can index any readable directory (bounded by the extension allowlist, the dotfile skip and a 450,000-chunk cap) and clear any tracked index. Evidence: [packages/core/src/utils/ignore-matcher.ts:20](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/core/src/utils/ignore-matcher.ts#L20); [packages/core/src/context.ts:860](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/core/src/context.ts#L860) (verified)
  - *To reach the next level:* Operations are not scoped to a project or workspace.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

The server never executes code. It parses source files with bundled tree-sitter grammars and sends text to the embedding provider and vector database; there is no subprocess, eval or dynamic code path in the server or core library.

- **Structural absence:** searched `rg -n 'child_process|execSync|spawn\(|eval\(|new Function|vm\.'` in `packages/mcp/src packages/core/src` → 0 hits (No subprocess, eval or VM usage in the MCP server or core library.)

### C5 Untrusted input blast radius — 0.10 (high)

Search results return raw code chunks from whatever repositories were indexed, which may include cloned third-party code containing instructions aimed at the model. Results carry a file location but no untrusted marking, and they are mixed with the server's own guidance text; the tool descriptions also contain directives to the model. On the other side, the server has no outbound channel an attacker can choose (it only talks to the operator's configured embedding provider and vector database) and its destructive action only removes rebuildable index data, but it can put any readable code into the host's context, where the host's other tools could leak it.

- **S L0:** Tool descriptions and outputs include directives to the model, and returned code is plain text inside a formatted message. Evidence: [packages/mcp/src/index.ts:95](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/index.ts#L95); [packages/mcp/src/handlers.ts:806-809](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/handlers.ts#L806-L809) (verified)
  - *To reach the next level:* Structured output separating returned content from server text is missing.
- **C L0:** No source is distinguished: indexed content from any repository returns with the same standing as server messages. Evidence: searched `rg -n -i 'untrusted|provenance'` in `packages/mcp/src packages/core/src` → 0 hits (No provenance or untrusted flag on returned content.) (verified)
  - *To reach the next level:* Untrusted marking on any returned content is missing.
- **D L0:** There is no control to turn on. Evidence: [packages/mcp/src/handlers.ts:816](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/handlers.ts#L816) (verified)
  - *To reach the next level:* No default mode that drops a Rule-of-Two leg.
- **B L2:** A hijacked host can pull any indexed or indexable code into its context and clear indexes, but this server offers no attacker-chosen outbound channel and no irreversible action beyond uploads to the operator's own providers. Evidence: [packages/mcp/src/handlers.ts:801-810](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/handlers.ts#L801-L810) (verified)
  - *To reach the next level:* Sensitive data remains reachable; a mode without index or clear operations is not offered.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.30 (high)

The vector index is a persistent retrieval store: whatever indexed files contain, including instructions planted in third-party code, is returned in later sessions, and background sync re-indexes every tracked codebase every five minutes by default without review. Each codebase gets its own collection, and the local tracking file and settings live in the user's home directory, not the workspace. Workspace ignore files can only change which files are indexed, not security settings. The local tracking list is also reconciled with the collections found in the vector-database account, so codebases discovered there are added and then re-synced without a user decision. Entries can be purged with clear_index.

- **S L1:** Index contents are stored and refreshed without validation, and returned with a file location but no untrusted marking; codebases discovered in the vector database are adopted into the local tracking list automatically. Evidence: [packages/mcp/src/handlers.ts:294-301](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/handlers.ts#L294-L301); [packages/mcp/src/sync.ts:10](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/sync.ts#L10) (verified)
  - *To reach the next level:* Provenance-tagged results presented as data and a user decision before adopting remote entries are missing.
- **C L1:** Only the default ignore and extension rules shape what enters the index; the tracking snapshot and the remote reconciliation have no control. Evidence: [packages/mcp/src/sync.ts:187-219](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/sync.ts#L187-L219) (verified)
  - *To reach the next level:* Controls on the tracking list and on background re-indexing are missing.
- **D L2:** Each codebase is stored in its own collection derived from its path, and the model cannot choose collection names; isolation across people sharing one vector-database account is not enforced. Evidence: [packages/core/src/context.ts:289](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/core/src/context.ts#L289) (verified)
  - *To reach the next level:* Per-user isolation within a shared vector-database account is missing.
- **B L1:** Poisoned code persists in the index across the user's sessions and is fed to a coding agent that can act on it, though it can be purged with clear_index. Evidence: [packages/mcp/src/sync.ts:303-308](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/sync.ts#L303-L308) (verified)
  - *To reach the next level:* Retrieved content is not session-scoped or reviewed before reuse.
- **Cap:** none

### C7 Third-party extensions — 1.00 (high)

The server loads no plugins, launches no other MCP servers and installs nothing at runtime; it only uses its fixed npm dependencies (tree-sitter grammars, the Milvus SDK and embedding SDKs). The README's npx ...@latest launch is the host's supply-chain choice, not something the server does.

- **Structural absence:** searched `rg -n -i 'import\(|require\('` in `packages/mcp/src packages/core/src` → 14 hits (All 14 hits are static requires of bundled tree-sitter grammars, local splitter modules and the os module; nothing is loaded from a runtime-chosen source.)

### C8 Secrets & sensitive-data protection — 0.38 (high)

API keys come from environment variables or a plaintext ~/.context/.env file, and startup logs only show whether each key is set, never its value. Dotfiles and .env files are always skipped during indexing, which keeps the most common secret files away from the embedding provider and the index. There is no telemetry. Source code itself, including any secrets inside it, is sent to the embedding provider and stored in the vector database by design, there is no secret scanning, and the debug-level logs on stderr include search queries and file paths. Both keys are long-lived and the vector-database key is often account-wide.

- **S L1:** Secrets come from env vars or a plaintext home-directory file; logging prints only key presence, and there is no redaction helper. Evidence: [packages/core/src/utils/env-manager.ts:10](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/core/src/utils/env-manager.ts#L10); [packages/mcp/src/config.ts:208](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/config.ts#L208); searched `rg -n -i 'redact|mask'` in `packages/mcp/src packages/core/src` → 0 hits (No redaction code.) (verified)
  - *To reach the next level:* Type-level masking or log redaction and restrictive permissions on the stored key file are missing.
- **C L2:** Logs show only key presence, and the index (and so the model and the embedding provider) never receives dotfiles or .env files; secrets embedded in ordinary source files are not filtered. Evidence: [packages/core/src/utils/ignore-matcher.ts:20](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/core/src/utils/ignore-matcher.ts#L20); [packages/core/src/context.ts:100](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/core/src/context.ts#L100) (verified)
  - *To reach the next level:* Model-bound and provider-bound content is not scanned for secrets.
- **D L2:** No telemetry exists and logs carry no key material, though debug logging of queries and paths is always on. Evidence: searched `rg -n -i 'telemetry|posthog|sentry'` in `packages/mcp/src packages/core/src` → 0 hits (No telemetry or crash reporting.); [packages/mcp/src/index.ts:9](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/index.ts#L9) (verified)
  - *To reach the next level:* Debug logging cannot be turned down and the dotfile filter is not paired with secret scanning.
- **B L1:** If a key is exposed, it is long-lived; the vector-database key recommended by the README is an account-level Personal Key. Evidence: [README.md:44](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/README.md#L44) (verified)
  - *To reach the next level:* Keys are not scoped to one cluster or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.33 (high)

The only record is free-text console output redirected to stderr: index starts, search queries and clear operations are all logged, but without structure, caller identity or a durable file of the server's own. Whether anything is kept depends on the host capturing stderr.

- **S L1:** Unstructured console logs of tool activity (search queries, clear operations, indexing progress). Evidence: [packages/mcp/src/handlers.ts:734-736](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/handlers.ts#L734-L736); [packages/mcp/src/handlers.ts:907](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/handlers.ts#L907) (verified)
  - *To reach the next level:* A structured per-call record with arguments, status and timestamp is missing.
- **C L2:** Every tool and the background sync emit log lines. Evidence: [packages/mcp/src/index.ts:8-10](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/index.ts#L8-L10) (verified)
  - *To reach the next level:* Records are not structured and do not cover configuration changes.
- **D L1:** Logging is on by default but goes only to stderr; the server keeps no audit file. Evidence: searched `rg -n -i 'appendFile|createWriteStream|audit'` in `packages/mcp/src packages/core/src` → 0 hits (No log or audit file writer.) (verified)
  - *To reach the next level:* A record the server writes outside the host's control is missing.
- **B L1:** Logging is best effort; nothing durable is guaranteed. Evidence: [packages/mcp/src/index.ts:8-10](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/index.ts#L8-L10) (verified)
  - *To reach the next level:* Records are not flushed durably per action.
- **Cap:** none

### C10 Limits & kill switch — 0.33 (high)

The server enforces some bounds of its own: search returns at most 50 results with snippets truncated to 5,000 characters, indexing stops at 450,000 chunks, and clear_index cancels an in-flight indexing run before dropping the collection. There are no rate limits, the directory walk and per-call embedding requests have no server-set timeouts, and background sync re-indexes every tracked codebase every five minutes by default, spending embedding credits with no budget.

- **S L2:** Server-enforced caps exist on search size, snippet length and total chunks, and indexing is cancellable. Evidence: [packages/mcp/src/handlers.ts:765](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/handlers.ts#L765); [packages/mcp/src/handlers.ts:803](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/handlers.ts#L803); [packages/mcp/src/handlers.ts:916](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/handlers.ts#L916) (verified)
  - *To reach the next level:* Caps on every operation plus concurrency or rate limits are missing.
- **C L1:** Caps apply to search and indexing, but background sync and the directory walk are unbounded. Evidence: [packages/mcp/src/sync.ts:303-308](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/sync.ts#L303-L308); searched `rg -n -i 'rateLimit|rate_limit|RateLimit'` in `packages/mcp/src packages/core/src` → 0 hits (No rate limiting.) (verified)
  - *To reach the next level:* Background sync and file traversal have no limits.
- **D L1:** Defaults are very large (450,000 chunks) and background sync is on by default. Evidence: [packages/core/src/context.ts:860](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/core/src/context.ts#L860); [packages/mcp/src/sync.ts:15-19](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/mcp/src/sync.ts#L15-L19) (verified)
  - *To reach the next level:* Sensible per-run defaults are missing.
- **B L1:** A runaway can embed hundreds of thousands of chunks and keep re-syncing indefinitely; stopping is only possible per codebase or by killing the process. Evidence: [packages/core/src/context.ts:915-916](https://github.com/zilliztech/claude-context/blob/6fc318b4e3ce58e2898b00a9c3538ead9e24dee5/packages/core/src/context.ts#L915-L916) (verified)
  - *To reach the next level:* Tight spend ceilings are missing.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Indexed code from any repository returned as plain text (packages/mcp/src/handlers.ts:801-810) · [B] sensitive data/systems: Any directory the OS user can read can be indexed (packages/mcp/src/handlers.ts:352-355); service keys held by the server (packages/mcp/src/config.ts:173-186) · [C] state change / egress: index_codebase uploads file contents to the configured embedding provider and vector database; clear_index drops collections (packages/mcp/src/handlers.ts:928) · Same default session? Yes

## Highest-impact improvements
1. Add readOnlyHint to search_code and get_indexing_status and destructiveHint to clear_index and index_codebase. (C2 S L0→L2, +0.150 before caps; Playbook 5)
2. Add an operator-configured list of allowed roots and reject index or search paths that resolve outside it. (C3 S L2→L3, +0.075 before caps; Playbook 3)
3. Return results as structured content with source path and an untrusted flag, and drop directives from tool descriptions. (C5 S L0→L2, +0.150 before caps; Playbook 1)
4. Write a structured JSON line per tool call (tool, arguments, status, timestamp) to a log file in ~/.context. (C9 S L1→L2, +0.075 before caps; Playbook 1 step 3)
5. Ask before adopting codebases discovered in the vector database and make background sync opt-in. (C6 S L1→L2, +0.075 before caps; Playbook 2)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scope is the MCP server (packages/mcp) and the core library it uses (packages/core); the VS Code and Chrome extensions were not scored.
- No release tag points at the pinned commit (git describe: v0.1.11-30-g6fc318b); package.json says 0.1.15.
- Behaviour of the Milvus, OpenAI and other SDKs (timeouts, retries) was not read in the dependencies.
- Zilliz Cloud and embedding-provider server-side behaviour (key scopes, retention, spend limits) was not examined.
- AGENTS.md and CLAUDE.md contain contributor guidance for coding agents; no text aimed at steering reviewers or scores was found.
