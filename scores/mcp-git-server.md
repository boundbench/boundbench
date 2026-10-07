# Defense-in-Depth Score: MCP Git server

**Repo:** https://github.com/modelcontextprotocol/servers (`src/git`) · **Commit:** `f46d9578190b476b3501923ea8977d899e8db2cb` · **Reviewed:** 2026-10-03
**What it is:** Reference MCP server for local git repo read/manipulation
**Category:** Coding
**Scored configuration:** stdio server launched with `uvx mcp-server-git` and no flags (CLI default: no --repository restriction), as in the README's VS Code one-click install and Zed examples.
**Agent surface (default):** code execution yes · filesystem write yes · network egress no · external credentials no · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 3.9 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L0 | L2 | 0.17 | — | **0.17** | High |
| C2 | Approval gates | L2 | L3 | L3 | L2 | 0.62 | — | **0.62** | Medium |
| C3 | Tool & action scoping | L2 | L3 | L0 | L2 | 0.47 | — | **0.47** | High |
| C4 | Code-execution isolation | L2 | L3 | L0 | L0 | 0.38 | G1 | **0.38** (alt) | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L2 | 0.17 | — | **0.17** | Medium |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L2 | 0.35 | — | **0.35** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L4 | 0.45 | — | **0.45** | High |
| C9 | Audit & traceability | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |

Controls where a risk surface exists: 2.88 / 9.0 (32%); 1 criterion scored SA (surface absent).

A small, narrowly built git server: no shell, no network tools, no credentials, accurate read/write annotations on every tool, and careful checks against flag injection and path traversal. Its weak points are defaults and visibility: with no --repository flag it can act on every git repository the user owns, it keeps no record of what it did, and it bounds none of its output. The dominant risk is indirect: git_commit and git_checkout run the repository's git hooks on the host as the user, so a repository whose hooks execute tracked code can turn a routine commit into arbitrary code execution.

## Critical gaps
- git_commit and git_checkout run the repository's git hooks on the host as the OS user with the full environment; nothing isolates them or disables hooks (impact inferred from GitPython/git behaviour). (ASI05; C4) — [src/git/src/mcp_server_git/server.py:128-130](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L128-L130); [src/git/src/mcp_server_git/server.py:197-204](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L197-L204)

## Criterion details

### C1 Identity & least privilege — 0.17 (high)

The server holds no credentials of its own and never forwards tokens, but it runs with the full authority of the OS user who launched it and does nothing to narrow that. Its only authorization check is an optional repository restriction; with the CLI default (no --repository) every git repository the user can reach on disk is fair game. Git subprocesses inherit the full environment, so any git credential helpers or SSH agents configured for the user remain reachable by git and its hooks. Because the server exposes no push, fetch, or network tool, a hijack is limited to local git writes.

- **S L0:** Runs as the launching OS user with no dedicated or narrowed identity; the only restriction (allowed repository) is a path allowlist credited under C3 and is unset by default. — [src/git/src/mcp_server_git/server.py:235-238](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L235-L238); [src/git/src/mcp_server_git/__init__.py:8](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/__init__.py#L8); searched `rg -n -i 'env=|environ|credential'` in `src/git/src` → 0 hits (no credential handling or subprocess environment control in server code) (verified)
  - *To reach the next level:* No narrowing of ambient authority: L1 needs a dedicated, even if broad, identity or scoped credential.
- **C L1:** When --repository is set, every tool call passes validate_repo_path before git.Repo is opened, but git subprocesses (via GitPython) inherit the full os.environ. — [src/git/src/mcp_server_git/server.py:472-478](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L472-L478); searched `rg -n 'validate_repo_path'` in `src/git/src` → 2 hits (definition and single call site in call_tool, which all 12 tools pass through) (verified)
  - *To reach the next level:* Git subprocesses and hooks receive the full user environment; L2 needs every tool to run under a scoped identity.
- **D L0:** --repository has no default, so a fresh install exposes every repository the OS user can access. — [src/git/src/mcp_server_git/__init__.py:8](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/__init__.py#L8); [src/git/src/mcp_server_git/server.py:235-238](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L235-L238) (verified)
  - *To reach the next level:* Least privilege requires manual hardening (passing --repository); L1 needs a narrower default.
- **B L2:** If authorization fails, the attacker gets local git writes (stage, commit, branch, checkout, unstage) on any repository the user can access; no push, fetch, or remote tool exists. — [src/git/src/mcp_server_git/server.py:96-109](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L96-L109); searched `rg -n -i 'push|fetch|pull|clone|remote'` in `src/git/src` → 2 hits (both hits are git_branch's 'remote' listing option (field description and case label), not network operations) (verified)
  - *To reach the next level:* Writes reach every local repository by default; L3 needs scope to one project with mostly read access.
- **Cap:** none

### C2 Approval gates — 0.62 (medium)

As a tool server, the git server can't approve anything itself, but it gives the host what it needs to gate actions: reads and writes are separate tools and all twelve carry accurate read-only/destructive annotations that nothing at runtime can change. It offers no dry-run or preview for its write operations and no server-enforced read-only mode. Write actions are local and mostly reversible through git itself, but committing and checking out run whatever git hooks the repository has installed, which the approver is not told about.

- **S L2:** Separate read and write tools; every tool carries ToolAnnotations with readOnlyHint/destructiveHint that match its behaviour (git_reset marked destructive). — [src/git/src/mcp_server_git/server.py:373-383](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L373-L383); searched `rg -n -i 'dry_run|dry-run|read_only|readonly|confirm'` in `src/git/src` → 12 hits (all 12 hits are readOnlyHint annotations; no dry-run or confirmation mode) (verified)
  - *To reach the next level:* No preview or dry-run for write operations (e.g. which files git_add '.' would stage); L3 requires one.
- **C L3:** All 12 tools are annotated in list_tools and the single dispatcher rejects unknown tool names; the tools marked read-only only read. — [src/git/src/mcp_server_git/server.py:304-317](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L304-L317); [src/git/src/mcp_server_git/server.py:580-581](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L580-L581) (verified)
  - *To reach the next level:* Hook execution inside git_commit/git_checkout isn't signalled to the host; L4 also needs parsed argument-level policy.
- **D L3:** Annotations are hard-coded in the server and no flag, env var, or tool input can change them. — [src/git/src/mcp_server_git/server.py:351-361](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L351-L361) (verified)
  - *To reach the next level:* No server-enforced read-only mode or confirmation step the host must complete.
- **B L2:** Approved writes are local git operations recoverable via git (reset is index-only), but git_commit runs the repository's pre-commit/commit-msg/post-commit hooks (GitPython default skip_hooks=False, read in GitPython 3.1.54 git/index/base.py:1136-1177) and checkout runs post-checkout, so a wrongly approved call can run arbitrary hook code. — [src/git/src/mcp_server_git/server.py:128-130](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L128-L130); [src/git/src/mcp_server_git/server.py:155-157](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L155-L157); [src/git/uv.lock:235-236](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/uv.lock#L235-L236) (inferred)
  - *To reach the next level:* No checkpoint or dry-run, and hooks run unannounced; L3 needs rollback for all state the tools touch.
- **Cap:** none

### C3 Tool & action scoping — 0.47 (high)

The tools are narrow git operations rather than a generic git or shell passthrough, and arguments get real checks: refs must resolve in the repository, values starting with '-' are rejected to stop flag injection, files passed to git_add must resolve inside the repository, and the MCP SDK validates argument types against each tool's schema. The repository-path allowlist resolves symlinks properly, but it is off unless the operator passes --repository, and the advertised MCP 'roots' list is never enforced. Numeric arguments such as log count and diff context are unbounded, and every write tool is always enabled.

- **S L2:** Resolved-path containment for repo_path and git_add files, rev_parse ref validation, and '-' prefix rejection; but max_count and context_lines have no bounds and flag-injection defense is a prefix denylist. — [src/git/src/mcp_server_git/server.py:240-253](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L240-L253); [src/git/src/mcp_server_git/server.py:139-152](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L139-L152); [src/git/src/mcp_server_git/server.py:120-126](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L120-L126); [src/git/src/mcp_server_git/server.py:50-52](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L50-L52) (verified)
  - *To reach the next level:* No numeric bounds (max_count, context_lines) and option rejection is a '-' prefix check; L3 needs allowlist validation with bounds on every argument.
- **C L3:** repo_path validation sits in the single call_tool dispatcher every tool uses; each ref/branch/path argument has its own check; SDK schema validation covers types. — [src/git/src/mcp_server_git/server.py:472-478](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L472-L478); [src/git/src/mcp_server_git/server.py:256-261](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L256-L261) (verified)
  - *To reach the next level:* Per-argument checks are hand-written per tool, not one policy layer new tools inherit; the MCP roots list (list_repos) is defined but never enforced.
- **D L0:** All write tools are always exposed with no tool selection or read-only mode, and the repository restriction is off by default. — [src/git/src/mcp_server_git/server.py:304-439](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L304-L439); [src/git/src/mcp_server_git/server.py:235-238](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L235-L238); [src/git/src/mcp_server_git/server.py:441](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L441) (verified)
  - *To reach the next level:* Write tools cannot be disabled in the server; L1 needs dangerous tools to be individually disableable.
- **B L2:** Operations are narrow and mostly reversible (no push, delete, or hard reset), but default reach is every repository the user can open and git_add '.' stages the whole tree. — [src/git/src/mcp_server_git/server.py:132-134](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L132-L134); [src/git/src/mcp_server_git/server.py:235-238](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L235-L238) (verified)
  - *To reach the next level:* No quantity bounds and machine-wide reach by default; L3 needs scoped, quantity-bounded operations.
- **Cap:** none

### C4 Code-execution isolation — 0.38 (high)

The server never runs a shell or evaluates model text: it calls git through GitPython with argument lists. But git_commit and git_checkout run the repository's installed git hooks, and with common setups (pre-commit framework, husky-style tracked hooks directories) those hooks execute code that comes from the repository's working tree, on the host, as the user, with the full environment. There is no isolation in the default uvx/pip install. The optional Docker image adds a plain container, but it runs as root, and the README's leading Docker example mounts the user's whole home directory.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Default install runs git and any hooks it triggers as a same-user host process; no sandbox code exists. — [src/git/src/mcp_server_git/server.py:128-130](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L128-L130); [src/git/src/mcp_server_git/server.py:197-204](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L197-L204); searched `rg -n -i 'sandbox|seccomp|setuid|subprocess|Popen'` in `src/git/src` → 0 hits (no isolation primitive in server code) (verified)
    - *To reach the next level:* No isolation; L1 would at least need hooks disabled (skip_hooks=True / --no-verify) or filtered.
  - **C L0:** Neither the commit path nor the checkout path is isolated or has hooks disabled. — [src/git/src/mcp_server_git/server.py:128-130](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L128-L130); [src/git/src/mcp_server_git/server.py:197-204](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L197-L204) (verified)
    - *To reach the next level:* No path is sandboxed; L1 needs the main exec path (hook-triggering git calls) contained.
  - **D L0:** Isolation exists only as the separately built Docker image; the default uvx/pip entry point runs on the host. — [src/git/README.md:134-138](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/README.md#L134-L138); [src/git/pyproject.toml:29-30](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/pyproject.toml#L29-L30) (verified)
    - *To reach the next level:* Isolation is off by default; L1 needs it on by default.
  - **B L0:** Hook code runs as the OS user with the full environment (GitPython copies os.environ, cmd.py:1375-1385 in 3.1.54), so it reaches the home directory, credentials, and network. — [src/git/src/mcp_server_git/server.py:128-130](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L128-L130); [src/git/uv.lock:235-236](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/uv.lock#L235-L236) (inferred)
    - *To reach the next level:* Host-equivalent; L1 needs at least a scrubbed environment or limited mounts.
- **opt-in Docker image (mcp/git)** (alt; raw 0.38, cap G1 → 0.38) ← counted
  - **S L2:** Stock python slim container; a non-root 'app' user is created but no USER directive is set, so the server and hooks run as root inside. — [src/git/Dockerfile:33-42](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/Dockerfile#L33-L42) (verified)
    - *To reach the next level:* Not hardened: runs as root with default capabilities; L3 needs non-root, dropped capabilities, no-new-privileges.
  - **C L3:** Every git invocation and hook triggered by the server runs inside the container; there is no host fallback. — [src/git/Dockerfile:42](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/Dockerfile#L42) (verified)
    - *To reach the next level:* No fail-closed guarantee for processes spawned by hooks beyond the container itself; L4 needs explicit coverage of all spawned processes.
  - **D L0:** Docker use is a separate, documented opt-in configuration. — [src/git/README.md:143-155](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/README.md#L143-L155) (verified)
    - *To reach the next level:* Off by default.
  - **B L0:** README's leading Docker example bind-mounts the user's entire home directory read-write with default container networking. — [src/git/README.md:152](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/README.md#L152) (verified)
    - *To reach the next level:* Home directory mounted; L1 needs at most a broad non-home mount without credentials.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.
- **Notes:** Hook execution is git/GitPython behaviour the server does not disable; whether attacker-controlled code runs depends on the user's hook setup (e.g. pre-commit framework or core.hooksPath pointing at a tracked directory).

### C5 Untrusted input blast radius — 0.17 (medium)

Everything the server returns comes straight from the repository: commit messages, diffs, file contents, branch names, authored by anyone who has contributed. It hands this back as plain text with a short label and no marker that it is untrusted, and attacker-written commit messages are inlined verbatim in git_log. The server itself has no network egress and no irreversible action, so a hijacked host can use it only for local, reversible git changes, plus whatever the repository's hooks do when it commits. It offers no read-only or no-egress mode a host could use to break the Rule of Two.

- **S L1:** Outputs are plain TextContent strings with fixed prefixes; tool descriptions contain no directives, but returned content carries no provenance or untrusted flag. — [src/git/src/mcp_server_git/server.py:174-180](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L174-L180); [src/git/src/mcp_server_git/server.py:483-486](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L483-L486); searched `rg -n -i 'untrusted|provenance|structuredContent|outputSchema'` in `src/git/src` → 0 hits (verified)
  - *To reach the next level:* No structured separation of returned content from metadata; L2 needs structured outputs.
- **C L0:** No returned source (log, show, diff, status, branch list) is distinguished as untrusted. — [src/git/src/mcp_server_git/server.py:208-233](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L208-L233) (verified)
  - *To reach the next level:* No source is marked; L1 needs at least one.
- **D L0:** There is no untrusted-input control to enable. — searched `rg -n -i 'untrusted|provenance|structuredContent|outputSchema'` in `src/git/src` → 0 hits (verified)
  - *To reach the next level:* No control exists; L1 needs one on by default.
- **B L2:** The server offers no egress and no irreversible action, so unattended damage through its own tools is limited to reversible local git changes; hooks triggered by git_commit could do more (inferred, setup-dependent). — [src/git/src/mcp_server_git/server.py:96-109](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L96-L109); [src/git/src/mcp_server_git/server.py:128-130](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L128-L130) (inferred)
  - *To reach the next level:* No server-side read-only mode; L3 needs only reversible actions under approval or no sensitive data reachable.
- **Cap:** none
- **Notes:** G1 not applied: there is no opt-in untrusted-input control; S L1 credits plain, directive-free output, which is always the behaviour.

### C6 Memory, context & configuration integrity — 0.35 (high)

The server keeps no memory and loads no configuration or instruction files of its own. The one persistence path a hijacked model controls is the repository itself: commit messages and branch names it writes stay in the history and come back, unmarked, through git_log, git_show, and git_branch in later sessions. That history is easy for a person to inspect but the server offers no way to remove what was written. Git itself still honours repository config, attributes, and hooks, but the server neither adds nor restricts that.

- **S L1:** Model-written commit messages and branch names persist unvalidated and are re-returned as plain output. — [src/git/src/mcp_server_git/server.py:128-130](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L128-L130); [src/git/src/mcp_server_git/server.py:183-195](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L183-L195); searched `rg -n -i 'dotenv|AGENTS|CLAUDE|config'` in `src/git/src` → 2 hits (neither hit loads a config or instruction file) (verified)
  - *To reach the next level:* No provenance on persisted content; L2 needs entries presented with provenance as data.
- **C L1:** No path controls what is persisted or how it returns; the only store is git history. — [src/git/src/mcp_server_git/server.py:174-180](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L174-L180) (verified)
  - *To reach the next level:* No store is controlled; L2 needs the main store (history output) to carry provenance or validation.
- **D L2:** Single-user local process; data lives in the user's own repositories with no cross-user sharing in server code. — [src/git/src/mcp_server_git/server.py:584-585](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L584-L585) (verified)
  - *To reach the next level:* Isolation is by the OS, not by namespaces the server enforces.
- **B L2:** Poisoned history persists across sessions but only influences text returned to the model, and is visible in ordinary git tooling. — [src/git/src/mcp_server_git/server.py:159-181](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L159-L181) (verified)
  - *To reach the next level:* No purge or review step offered by the server; L3 needs easy purge or session scope.
- **Cap:** none

### C7 Third-party extensions — 1.00 (high)

The server loads no plugins, downloads no tools, and launches no other servers: its only runtime dependencies are pinned in the lockfile and installed by the user. There is no extension surface to compromise.

- **Structural absence:** searched `rg -n -i 'importlib|entry_points|__import__|pip install|npx|plugin|load_module'` in `src/git/src` → 0 hits (no dynamic loading or install path in server code)

### C8 Secrets & sensitive-data protection — 0.45 (high)

The server accepts no API keys or tokens, sends nothing to telemetry, and logs almost nothing (no tool arguments or outputs), so there is little for it to leak. It also has no redaction: secrets committed to a repository flow back to the model unfiltered through diffs and git_show, and git and its hooks receive the user's full environment.

- **S L1:** No secret handling or redaction exists; nothing the server holds needs masking, but repository content is returned unfiltered. — searched `rg -n -i 'secret|token|password|redact|mask|sentry|telemetry'` in `src/git/src` → 0 hits (verified)
  - *To reach the next level:* No redaction on any path; L2 needs masking/log filters on main paths.
- **C L1:** Logs carry no tool payloads (one path effectively clean), but model-bound outputs and subprocess environments are unprotected. — searched `rg -n -i 'logger\.|logging\.'` in `src/git/src` → 8 hits (startup info/error, a debug line for MCP roots, and log-level setup; no tool-call logging) (verified)
  - *To reach the next level:* Model-bound outputs and subprocess environments unprotected; L2 needs logs and transcripts covered.
- **D L2:** No telemetry; default log level WARN; debug logs only the MCP roots list. — [src/git/src/mcp_server_git/__init__.py:14-20](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/__init__.py#L14-L20) (verified)
  - *To reach the next level:* No redaction to keep on; L3 needs always-on redaction.
- **B L4:** Ambient-only design: the server accepts no key material; residual exposure is repository secrets and the inherited environment available to git hooks. — searched `rg -n -i 'env=|environ|credential'` in `src/git/src` → 0 hits (no credential handling or subprocess environment control in server code) (verified)
- **Cap:** none

### C9 Audit & traceability — 0.00 (high)

The server keeps no record of the tool calls it executes: no arguments, results, or timestamps are logged at any level. Git's own reflog records some ref changes, but that is git's bookkeeping inside the repository, not an audit trail the server writes. After an incident, you would have to rely on the host's logs.

- **S L0:** No tool call is logged at any level. — searched `rg -n -i 'logger\.|logging\.'` in `src/git/src` → 8 hits (startup info/error, a debug line for MCP roots, and log-level setup; no tool-call logging) (verified)
  - *To reach the next level:* No record of tool calls; L1 needs at least unstructured logs of actions.
- **C L0:** Nothing is recorded for any tool. — [src/git/src/mcp_server_git/server.py:470-481](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L470-L481) (verified)
  - *To reach the next level:* No coverage; L1 needs the main tool path logged.
- **D L0:** There is no audit record to enable. — searched `rg -n -i 'logger\.|logging\.'` in `src/git/src` → 8 hits (startup info/error, a debug line for MCP roots, and log-level setup; no tool-call logging) (verified)
  - *To reach the next level:* No record exists to be on by default.
- **B L0:** Actions proceed with no record written. — [src/git/src/mcp_server_git/server.py:470-472](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L470-L472) (verified)
  - *To reach the next level:* No record at all; L1 needs best-effort logging.
- **Cap:** none

### C10 Limits & kill switch — 0.25 (high)

The server bounds almost none of its own work. git_log defaults to ten commits but the caller can raise it at will, and diffs, git_show output, status, and branch listings are unbounded in size. No git call has a timeout, and the server offers no rate limiting or cancellation of an in-flight operation beyond the host killing the process.

- **S L1:** Only caller-chosen limits (max_count, context_lines); no server-enforced output cap or timeout. — [src/git/src/mcp_server_git/server.py:166](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L166); searched `rg -n -i 'timeout|max_output|truncat|kill|cancel'` in `src/git/src` → 0 hits (verified)
  - *To reach the next level:* No server-enforced caps; L2 needs output-size caps or timeouts.
- **C L1:** The one limit applies only to git_log. — [src/git/src/mcp_server_git/server.py:159-172](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L159-L172) (verified)
  - *To reach the next level:* Diff, show, status, and branch have no limits.
- **D L1:** max_count defaults to 10 but the model can raise it without bound. — [src/git/src/mcp_server_git/server.py:50-52](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L50-L52) (verified)
  - *To reach the next level:* The model can raise the only default; L2 needs sensible operator-controlled defaults.
- **B L1:** A large diff or slow git operation runs to completion with no ceiling; stopping relies on the host terminating the stdio process. — [src/git/src/mcp_server_git/server.py:114-118](https://github.com/modelcontextprotocol/servers/blob/f46d9578190b476b3501923ea8977d899e8db2cb/src/git/src/mcp_server_git/server.py#L114-L118) (verified)
  - *To reach the next level:* No ceilings; L2 needs moderate output/time ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: commit messages, diffs and file contents returned by git_log/git_show/git_diff (server.py:174-180, 208-233) · [B] sensitive data/systems: any repository the OS user can open when --repository is unset (server.py:235-238) · [C] state change / egress: local commits, staging, branch creation and checkout (server.py:128-204); no network egress tool · Same default session? Yes

## Highest-impact improvements
1. Pass skip_hooks=True to index.commit and run checkout with hooks disabled (e.g. -c core.hooksPath=/dev/null), or make hook execution an explicit operator flag. — C4 S L0→L1, +0.075 before caps (Playbook 3, step 1)
2. Log every tool call (name, arguments, result status, timestamp) to stderr or a file outside the repository. — C9 S L0→L2, +0.150 before caps (Playbook 1, step 3)
3. Add a --read-only flag that drops commit/add/reset/branch/checkout tools from list_tools. — C3 D L0→L1, +0.050 before caps (Playbook 3, step 1)
4. Cap output size and max_count, and add a timeout to git subprocess calls. — C10 S L1→L2, +0.075 before caps (Playbook 3, step 3)
5. Enforce MCP roots (list_repos) when --repository is unset instead of allowing any path. — C1 D L0→L1, +0.050 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only (subpath src/git); nothing was executed, installed, or probed.
- Hook-execution findings (C2 B, C4, C5 B) rely on third-party behaviour: GitPython 3.1.54 IndexFile.commit (skip_hooks=False default, git/index/base.py:1136-1177) and os.environ inheritance (git/cmd.py:1375-1385), read from upstream GitPython source, plus git's post-checkout hook. Whether attacker-controlled code runs depends on the user's hook setup.
- MCP SDK input-schema validation (lowlevel Server.call_tool validate_input=True) was confirmed in mcp 1.28.1 source, not the locked 1.29.0.
- The scored default omits --repository (CLI parser default, VS Code and Zed examples); the Claude Desktop examples pass it, which would raise C1 D and C3 B/D-related reach.
- Repo is in an MIT -> Apache-2.0 license transition; license not scored.
- No reviewer-injection text found in src/git.
