# Defense-in-Depth Score: GitHub MCP Server

**Repo:** https://github.com/github/github-mcp-server · **Commit:** `f10e4e1f923d46b86f2e80e849aa74084c847184` (v1.14.0) · **Reviewed:** 2026-10-03
**What it is:** GitHub's official MCP server (repos, issues, PRs, Actions, security)
**Category:** Coding
**Scored configuration:** Local stdio server from the official Docker image with no flags: default toolsets (context, repos, issues, pull_requests, users, copilot), authenticated by the built-in OAuth login with its default scopes (including full 'repo') or by a user-supplied personal access token.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents yes · external communication yes

## Score: 6.4 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L2 | L2 | 0.50 | G1 | **0.50** (alt) | High |
| C2 | Approval gates | L2 | L3 | L3 | L2 | 0.62 | — | **0.62** | High |
| C3 | Tool & action scoping | L3 | L2 | L2 | L1 | 0.53 | — | **0.53** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L4 | L2 | L2 | L2 | 0.65 | G1 | **0.50** (alt) | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C9 | Audit & traceability | L1 | L2 | L2 | L1 | 0.38 | G1 | **0.38** (alt) | High |
| C10 | Limits & kill switch | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |

Controls where a risk surface exists: 3.35 / 7.0 (48%); 3 criteria scored SA (surface absent).

The server acts on GitHub with the user's own token, which by default carries full 'repo' control of every private repository the user can reach, and its default tool set can push files, merge pull requests, post comments and open issues with no confirmation step of its own. Content from any public issue, pull request or file flows to the model alongside those write tools and private-repo reads, so a prompt-injected agent can leak private code into a public comment unattended: the server's lockdown mode, information-flow labels and read-only mode would break that chain but are all off by default. Risk annotations are accurate and enforced by a test, repository deletion needs a typed human confirmation, and the token is never sent off GitHub hosts, but there is no audit record of tool calls unless verbose stdio logging is switched on.

## Critical gaps
- In the default configuration, content from any public issue, PR or file reaches the model in the same session that holds full 'repo' access to private repositories and unattended write tools (comments, issues, push_files), so a prompt injection can exfiltrate private code publicly; lockdown, IFC labels and read-only mode are all opt-in. (ASI01, ASI02, LLM01; C5) — [pkg/scopes/scopes.go:84-85](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/scopes/scopes.go#L84-L85); [pkg/github/issues.go:1369-1374](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/issues.go#L1369-L1374); [cmd/github-mcp-server/main.go:253](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/cmd/github-mcp-server/main.go#L253)

## Criterion details

### C1 Identity & least privilege — 0.50 (high)

The server uses one GitHub credential for everything: a personal access token, or by default an OAuth login that requests the full 'repo' scope (complete control of all the user's private repositories) plus packages, projects, gists and notifications. It narrows the default OAuth request by leaving out repository deletion, organization admin and workflow scopes, and it hides tools the token cannot use, but reads and writes share the same broad token and the server has no authorization layer of its own; GitHub's API is the only check. An opt-in GitHub App mode gives a dedicated, installation-scoped identity with short-lived tokens, which is the stronger option. In HTTP mode (not the scored default) the server forwards the client's bearer token to GitHub.

- **default configuration** (default; raw 0.17 → 0.17)
  - **S L0:** Default authority is the user's own token with the full 'repo' scope (OAuth default) or whatever scopes the user's PAT carries, shared by every read and write tool. — [pkg/scopes/scopes.go:84-85](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/scopes/scopes.go#L84-L85); [pkg/scopes/scopes.go:16-17](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/scopes/scopes.go#L16-L17); [cmd/github-mcp-server/main.go:49](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/cmd/github-mcp-server/main.go#L49) (verified)
    - *To reach the next level:* No dedicated or role-scoped identity by default; read and write tools are not given separate, narrower credentials.
  - **C L1:** All tools reach GitHub through the same host-restricted client built from the one credential; the only per-request authorization is GitHub's own, and the token-scope filter only hides tools. — [internal/ghmcp/server.go:157-166](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/internal/ghmcp/server.go#L157-L166); [internal/ghmcp/server.go:220-223](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/internal/ghmcp/server.go#L220-L223); [pkg/github/scope_filter.go:31-37](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/scope_filter.go#L31-L37) (verified)
    - *To reach the next level:* No server-side authorization layer that every tool passes through with a scoped identity.
  - **D L1:** The default OAuth request omits delete_repo, admin:org and workflow, but still includes full 'repo', and GITHUB_OAUTH_SCOPES / --oauth-scopes or a broader PAT widens it without warning. — [pkg/scopes/scopes.go:84-101](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/scopes/scopes.go#L84-L101); [cmd/github-mcp-server/main.go:142-147](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/cmd/github-mcp-server/main.go#L142-L147) (verified)
    - *To reach the next level:* Default is not read-only or near-minimal; write access is granted without explicit operator elevation.
  - **B L1:** A hijacked session can write to every repository, issue and pull request across all orgs the user belongs to, plus gists and packages. — [pkg/scopes/scopes.go:84-95](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/scopes/scopes.go#L84-L95); [pkg/github/tools.go:40-45](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/tools.go#L40-L45) (verified)
    - *To reach the next level:* Authority is not confined to one project or tenant, and writes are not limited to non-destructive operations.
- **opt-in GitHub App installation tokens** (alt; raw 0.50, cap G1 → 0.50) ← counted
  - **S L2:** GitHub App mode mints installation access tokens for a dedicated app identity whose permissions and repositories are set by the app installation; tokens expire and are refreshed. — [internal/githubapp/githubapp.go:131-139](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/internal/githubapp/githubapp.go#L131-L139); [internal/githubapp/githubapp.go:171-178](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/internal/githubapp/githubapp.go#L171-L178) (verified)
    - *To reach the next level:* The token request sends no permissions or repositories body, so the server never narrows the token below the installation's full permission set, and reads and writes share it.
  - **C L2:** Every tool uses the provider-backed token through the same transport. — [internal/ghmcp/server.go:157-166](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/internal/ghmcp/server.go#L157-L166); [cmd/github-mcp-server/main.go:159-165](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/cmd/github-mcp-server/main.go#L159-L165) (verified)
    - *To reach the next level:* No server-side per-request authorization layer.
  - **D L2:** Requires explicit app configuration; widening means changing the app's permissions in GitHub. — [cmd/github-mcp-server/main.go:54](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/cmd/github-mcp-server/main.go#L54) (verified)
    - *To reach the next level:* Not the default; nothing forces a read-only installation.
  - **B L2:** Blast radius is the repositories and permissions of one app installation, with tokens that expire within an hour. — [internal/githubapp/githubapp.go:161-164](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/internal/githubapp/githubapp.go#L161-L164) (verified)
    - *To reach the next level:* Installation permissions may still include broad write access across the installation's repositories.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.
- **Notes:** HTTP mode (`github-mcp-server http`) takes the client's Authorization bearer token and forwards it to GitHub (pkg/http/middleware/token.go); had that mode been scored, C1-PASSTHRU would apply.

### C2 Approval gates — 0.62 (high)

The host, not the server, decides what to approve, so this criterion rates the signals the server gives it. Every tool explicitly declares whether it is read-only, a test enforces that the declaration is present, reads and writes are separate tools, and the opt-in read-only mode drops every write tool. Repository deletion is the one action with a server-enforced human confirmation: the user must type the exact repository name and the server re-checks the repository's identity before deleting. Other destructive or hard-to-undo actions (merging, deleting files, pushing files, posting comments) have no preview or confirmation, and the interactive forms some clients show for creating issues and pull requests are not an enforced gate.

- **S L2:** Separate read and write tools with explicit, accurate readOnlyHint (destructive hint set on deletes); delete_repository additionally requires an elicited, typed confirmation bound to sealed state. — [pkg/github/repositories.go:768-772](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/repositories.go#L768-L772); [pkg/github/repositories.go:879-884](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/repositories.go#L879-L884); [pkg/github/pullrequests.go:1530-1535](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/pullrequests.go#L1530-L1535) (verified)
  - *To reach the next level:* No preview or dry-run for destructive operations other than repository deletion (merge, delete_file, push_files have none).
- **C L3:** Every registered tool must declare ReadOnlyHint explicitly (enforced by an AST test over the package), and the read-only filter applies to every tool via the shared inventory. — [pkg/github/tools_static_validation_test.go:27-31](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/tools_static_validation_test.go#L27-L31); [pkg/inventory/filters.go:37-40](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/inventory/filters.go#L37-L40) (verified)
  - *To reach the next level:* Make the MCP Apps form step for create/update tools an enforced gate.
- **D L3:** Risk annotations ship on every tool and no config, env var or workspace file changes them; tool title/description overrides cannot alter the hints. — [pkg/inventory/server_tool.go:121-124](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/inventory/server_tool.go#L121-L124) (verified)
  - *To reach the next level:* Read-only mode is off by default (--read-only defaults to false) and no confirmation comes from an authenticated principal except for repository deletion.
- **B L2:** Most default writes are reversible on GitHub (commits can be reverted, issues reopened, force is disabled on ref updates), but comments, issues and PRs notify people immediately and merges can trigger deployments. — [pkg/github/repositories.go:1457](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/repositories.go#L1457); [pkg/github/issues.go:1369-1374](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/issues.go#L1369-L1374) (verified)
  - *To reach the next level:* No checkpoint/preview for external actions such as comments and merges.
- **Cap:** none

### C3 Tool & action scoping — 0.53 (high)

Tools are narrow, purpose-built GitHub operations rather than a generic HTTP or GraphQL passthrough. Write tools validate file paths (no absolute paths, backslashes or '..'), refuse to overwrite an existing file without its current SHA, refuse to write through symlinks, and never force-update branches. Read tools take paths and repository names without the same validation, and numeric limits such as page size live only in the schema. The default tool set includes many write tools (push files, delete files, create repositories, merge pull requests), and nothing limits which repositories or organizations the tools may touch.

- **S L3:** Narrow tools with code-level validation on writes: relative-path checks rejecting traversal, SHA-guarded overwrites, symlink write blocking, and non-forced ref updates. — [pkg/github/repository_path.go:14-34](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/repository_path.go#L14-L34); [pkg/github/repositories.go:596-603](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/repositories.go#L596-L603); [pkg/github/repositories.go:570-572](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/repositories.go#L570-L572) (verified)
  - *To reach the next level:* Owner/repo names and numeric bounds are not validated in handler code, and there is no repository/organization allowlist.
- **C L2:** Path validation is applied only in the three file-writing tools; get_file_contents and other read tools take paths unvalidated. — searched `rg -n 'validateRelativePath\(' --glob '!*_test.go'` in `pkg` → 6 hits (3 call sites (create_or_update_file, delete_file, push_files), the definition, and 2 uses inside scope-challenge helpers); [pkg/github/repositories.go:1009](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/repositories.go#L1009) (verified)
  - *To reach the next level:* Read tools and other free-form arguments are not covered by a shared validation layer.
- **D L2:** Toolsets are selectable, but the default set includes repos, issues and pull_requests with their write tools. — [pkg/github/tools.go:40-45](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/tools.go#L40-L45); [cmd/github-mcp-server/main.go:247](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/cmd/github-mcp-server/main.go#L247) (verified)
  - *To reach the next level:* The default tool set is not read-only; write tools are not opt-in.
- **B L1:** A misused tool can write to any repository the token reaches, and push_files accepts an unbounded array of files per commit. — [pkg/github/repositories.go:1639](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/repositories.go#L1639) (verified)
  - *To reach the next level:* Not scoped to a project or bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

The server never runs model-supplied code or commands. The only process it starts is the system browser opener for the OAuth login URL, and its only template rendering is the static OAuth result page. Actions it can trigger on GitHub (assigning the Copilot coding agent, or workflow runs in the non-default actions toolset) execute on GitHub's infrastructure and are scored as consequential actions under approval gates and untrusted input.

- **Structural absence:** searched `rg -n 'os/exec|exec\.Command' --glob '!*_test.go'` in `cmd/github-mcp-server internal pkg` → 4 hits (all in internal/oauth/env.go: opening the OAuth authorization URL in the system browser, not model-controlled code); searched `rg -n 'text/template|html/template|goja|yaegi|starlark' --glob '!*_test.go'` in `cmd/github-mcp-server internal pkg` → 2 hits (internal/oauth/callback.go renders a static, auto-escaped OAuth result page)
- **Notes:** With a token that carries the 'workflow' scope (not in the default OAuth set), push_files/create_or_update_file can write .github/workflows files that GitHub Actions then runs remotely; that is remote execution on GitHub, outside this server's process.

### C5 Untrusted input blast radius — 0.50 (high)

The server returns issue, pull request, comment, commit and file content from any repository, including public ones anyone can write to, and the same default session holds private-repository read access and write tools that post comments, open issues and push files. Results are structured JSON with authors and URLs, and issue and comment bodies are stripped of invisible characters, but nothing marks content as untrusted by default. The server already contains the right defenses — a lockdown mode that withholds public content from non-collaborators, information-flow labels that mark each result as trusted/untrusted and public/private, and a read-only mode — but all three are opt-in. As shipped, a hijacked agent can copy private code into a public comment with no human involved.

- **default configuration** (default; raw 0.45, cap C5-WORSTCASE → 0.25)
  - **S L2:** Structured JSON results separate content from metadata (author, URLs), with invisible-character stripping on bodies; untrusted/provenance labels exist only behind a feature flag. — [pkg/github/minimal_types.go:817-818](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/minimal_types.go#L817-L818); [pkg/github/ifc_labels.go:20-22](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/ifc_labels.go#L20-L22) (verified)
    - *To reach the next level:* No untrusted flag or provenance label on returned content in the default configuration.
  - **C L2:** Sanitization covers issue/PR/comment/release/commit text, but file contents, code search and job logs are returned as-is. — searched `rg -n 'sanitize\.(Content|PlainText|Sanitize)\(' --glob '!*_test.go'` in `pkg/github` → 35 hits (all are field-level sanitization of titles/bodies/messages in minimal_types.go, issues.go, discussions.go, repositories.go, projects.go; none on file contents); [pkg/github/repositories.go:1009](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/repositories.go#L1009) (verified)
    - *To reach the next level:* File contents, code search results and Actions logs carry no sanitization or provenance flag.
  - **D L3:** Output structuring and sanitization are always on and cannot be disabled by config or content; in HTTP mode a request header can only enable lockdown, not disable it. — [pkg/github/minimal_types.go:817-818](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/minimal_types.go#L817-L818); [cmd/github-mcp-server/main.go:253](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/cmd/github-mcp-server/main.go#L253) (verified)
    - *To reach the next level:* The stronger controls (lockdown, IFC labels) are off by default, so nothing the server ships on by default flags untrusted content.
  - **B L0:** Default session combines untrusted public content, private-repo data via the 'repo' scope, and unattended egress/state change (comments, issues, push_files, PRs) with no server-side gate. — [pkg/scopes/scopes.go:84-85](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/scopes/scopes.go#L84-L85); [pkg/github/issues.go:1369-1374](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/issues.go#L1369-L1374); [pkg/github/repositories.go:1639](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/repositories.go#L1639) (verified)
    - *To reach the next level:* Exfiltration and irreversible actions after reading untrusted content are not forced through human approval or disabled.
- **opt-in lockdown mode + ifc_labels feature + read-only mode** (alt; raw 0.65, cap G1 → 0.50) ← counted
  - **S L4:** IFC labels mark each result trusted/untrusted and public/private, lockdown fails closed on content from non-collaborators in public repos, and read-only mode removes every write tool (dropping the state-change/egress leg). — [pkg/ifc/ifc.go:7-11](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/ifc/ifc.go#L7-L11); [pkg/github/lockdown.go:19-36](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/lockdown.go#L19-L36); [pkg/inventory/filters.go:37-40](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/inventory/filters.go#L37-L40) (verified)
  - **C L2:** IFC labels are attached by many read tools and lockdown covers issue and pull request reads, but lockdown does not cover files, commits or code search. — searched `rg -n 'attachStaticIFCLabel\(|attachRepoVisibilityIFCLabel\(' --glob '!*_test.go'` in `pkg/github` → 45 hits (includes 2 function definitions; remaining hits are call sites across ~18 tool files); [pkg/github/lockdown.go:23-36](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/lockdown.go#L23-L36) (verified)
    - *To reach the next level:* Lockdown does not cover every untrusted source (file contents, commits, code search).
  - **D L2:** All three are operator flags/feature flags, off by default; IFC labels are not even in the insiders set. — [cmd/github-mcp-server/main.go:247-253](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/cmd/github-mcp-server/main.go#L247-L253); [pkg/github/feature_flags.go:62-66](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/feature_flags.go#L62-L66) (verified)
    - *To reach the next level:* Not on by default.
  - **B L2:** With read-only mode no write or egress tool exists, but private repository data remains readable alongside untrusted content. — [pkg/inventory/filters.go:37-40](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/inventory/filters.go#L37-L40) (verified)
    - *To reach the next level:* Sensitive data still reachable from sessions that read untrusted content.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C6 Memory, context & configuration integrity — 1.00 (high)

The server keeps no memory between sessions: OAuth tokens, caches and lockdown results live only in process memory, and configuration comes from flags and environment variables set by the operator. In the scored Docker deployment the working directory is inside the image, so no workspace file is loaded. In binary (non-Docker) installs, configuration loading is not integrity-protected.

- **Structural absence:** searched `rg -n 'os\.(WriteFile|Create|OpenFile|MkdirAll)|AddConfigPath|godotenv|ReadInConfig|UserConfigDir|UserHomeDir' --glob '!*_test.go'` in `cmd/github-mcp-server internal pkg` → 11 hits (docs generators (generate_docs.go, feature_flag_docs.go), test snapshot helper (toolsnaps.go), operator log file (server.go x2), --export-translations dump, and the translations loader; nothing persists model-influenced state); searched `rg -n 'WORKDIR /server'` in `Dockerfile` → 1 hits (the default Docker image's working directory is image-controlled, so no workspace file is loaded in the scored deployment)
- **Notes:** Not structurally absent for binary installs: configuration loading is not integrity-protected.

### C7 Third-party extensions — 1.00 (high)

The server loads no third-party code at runtime: no plugins, no MCP servers of its own, no downloaded tools. Its UI resources are embedded into the binary at build time.

- **Structural absence:** searched `rg -n 'plugin\.Open|go-plugin|mcp\.NewClient|CommandTransport|npx|pip install|go install' --glob '!*_test.go'` in `cmd/github-mcp-server internal pkg` → 0 hits (no plugin loading, MCP client, or package install path); [pkg/github/ui_embed.go:17](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/ui_embed.go#L17)

### C8 Secrets & sensitive-data protection — 0.45 (high)

The GitHub token never reaches the model: tools have no way to read it, OAuth tokens are held only in memory, and the token is attached only to requests for the configured GitHub hosts, so a redirect cannot carry it elsewhere. There is no redaction layer, though: the opt-in command logging writes every request and response verbatim, and the non-default secret-scanning tools deliberately return leaked secret values to the model. Default telemetry is a no-op. The token itself is long-lived and broad (a PAT, or an OAuth token with 'repo').

- **S L2:** Credential kept in memory or env and confined by a host allowlist on the auth transport; no type-level masking or redaction filters. — [pkg/http/transport/bearer.go:42-46](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/http/transport/bearer.go#L42-L46); [internal/ghmcp/server.go:65-73](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/internal/ghmcp/server.go#L65-L73) (verified)
  - *To reach the next level:* No redaction of secrets before logs or model-bound results; no keychain or secret-manager integration.
- **C L2:** Token stays out of logs and model context by construction, but command logging records full payloads and secret-scanning results put secrets into model context; the browser-opener subprocess inherits the environment. — [pkg/log/io.go:35](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/log/io.go#L35); [pkg/github/secret_scanning.go:88](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/secret_scanning.go#L88); [internal/oauth/env.go:30](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/internal/oauth/env.go#L30) (verified)
  - *To reach the next level:* Model-bound tool results and verbose logs are not filtered for secrets.
- **D L2:** Payload logging is off by default and metrics are a no-op sink in stdio mode. — [cmd/github-mcp-server/main.go:249](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/cmd/github-mcp-server/main.go#L249); [internal/ghmcp/server.go:189](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/internal/ghmcp/server.go#L189) (verified)
  - *To reach the next level:* No always-on redaction: enabling command logging writes unredacted payloads.
- **B L1:** A leaked token is long-lived (PAT) or a user OAuth grant with 'repo' scope, though it is never reachable by the model. — [pkg/scopes/scopes.go:84-85](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/scopes/scopes.go#L84-L85); [cmd/github-mcp-server/main.go:49](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/cmd/github-mcp-server/main.go#L49) (verified)
  - *To reach the next level:* Default credentials are not narrowly scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.38 (high)

By default the server keeps no record of the tools it runs: it logs startup and authentication events only, and the per-tool logger it passes to handlers is never used. An opt-in command-logging flag writes the raw JSON-RPC traffic (every request and response) to stderr or a log file, which lets you reconstruct calls, but as unparsed byte chunks with no notion of who asked or who approved, written best-effort.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** No tool-call record in the default configuration; handlers never call the injected logger. — searched `rg -n 'Logger\(' --glob '!*_test.go'` in `pkg/github` → 5 hits (all are the interface declaration and two Logger() implementations in dependencies.go; no tool calls them); [internal/ghmcp/server.go:340](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/internal/ghmcp/server.go#L340) (verified)
    - *To reach the next level:* No structured per-call record (tool, arguments, status, timestamp).
  - **C L0:** Nothing about tool calls is recorded by default. — [internal/ghmcp/server.go:407-410](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/internal/ghmcp/server.go#L407-L410) (verified)
    - *To reach the next level:* No tool path is recorded by default.
  - **D L0:** Command logging is opt-in. — [cmd/github-mcp-server/main.go:249](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/cmd/github-mcp-server/main.go#L249) (verified)
    - *To reach the next level:* Recording is not on by default.
  - **B L0:** With no record, nothing survives an incident. — [internal/ghmcp/server.go:407-414](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/internal/ghmcp/server.go#L407-L414) (verified)
    - *To reach the next level:* No per-action record is written.
- **opt-in --enable-command-logging** (alt; raw 0.38, cap G1 → 0.38) ← counted
  - **S L1:** Raw stdin/stdout bytes are logged with timestamps via slog, chunked by read/write rather than per call. — [pkg/log/io.go:29-46](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/log/io.go#L29-L46) (verified)
    - *To reach the next level:* Not a structured per-call record and no actor attribution.
  - **C L2:** All JSON-RPC traffic on stdio passes through the logger, so every tool call and result is captured. — [internal/ghmcp/server.go:407-410](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/internal/ghmcp/server.go#L407-L410) (verified)
    - *To reach the next level:* No approvals, configuration or credential-use events.
  - **D L2:** Written to stderr or an operator-chosen file outside any workspace; the model has no local file tool, but the process can alter it. — [internal/ghmcp/server.go:328-334](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/internal/ghmcp/server.go#L328-L334) (verified)
    - *To reach the next level:* Not written by a component separate from the server process.
  - **B L1:** slog writes per chunk but handler errors are ignored and actions proceed regardless. — [pkg/log/io.go:45-46](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/log/io.go#L45-L46) (verified)
    - *To reach the next level:* Logging failures are not surfaced and records are not durable per action.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch — 0.38 (high)

Some of the server's work is bounded: Actions log downloads are capped by a 5,000-line content window and default to the last 500 lines, list tools default to 30 results, and GitHub's own API rate limits bound a runaway. But GitHub API calls have no timeouts of their own, the log download ignores cancellation, page-size limits live only in the schema, and the server has no rate limiting. Assigning the Copilot coding agent starts remote work that continues regardless of anything the server or host does next.

- **S L2:** Server-enforced cap on job-log size (min of tail_lines and content window) and default page sizes; no timeouts or rate limits. — [pkg/github/actions.go:182](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/actions.go#L182); [cmd/github-mcp-server/main.go:252](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/cmd/github-mcp-server/main.go#L252); [pkg/github/actions.go:172](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/actions.go#L172) (verified)
  - *To reach the next level:* No caps on every operation (file reads, search) and no concurrency or rate limits.
- **C L1:** Limits apply to individual tools only; delegated Copilot agent sessions run outside any server budget. — [pkg/github/copilot.go:155-170](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/copilot.go#L155-L170); [internal/ghmcp/server.go:90-93](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/internal/ghmcp/server.go#L90-L93) (verified)
  - *To reach the next level:* No HTTP timeouts on tool calls and delegated work does not count against any limit.
- **D L2:** Sensible defaults (500 log lines, 5,000-line window, 30 per page) that the operator can change; the model cannot exceed the content window. — [pkg/github/actions.go:731-734](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/actions.go#L731-L734); [pkg/github/params.go:421](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/params.go#L421) (verified)
  - *To reach the next level:* The model can still escape limits by delegating to the Copilot agent, and no hard ceiling exists.
- **B L1:** Copilot agent sessions continue after the host stops, and downloads without context cannot be cancelled; GitHub API rate limits are the main external ceiling. — [pkg/github/copilot.go:158-160](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/copilot.go#L158-L160); [pkg/github/actions.go:172](https://github.com/github/github-mcp-server/blob/f10e4e1f923d46b86f2e80e849aa74084c847184/pkg/github/actions.go#L172) (verified)
  - *To reach the next level:* Stopping does not cancel delegated agent work or all in-flight requests.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: issues, PRs, comments and files from any public repository (pkg/github/minimal_types.go:817-818, pkg/github/repositories.go:954) · [B] sensitive data/systems: private repositories via the default 'repo' OAuth scope or PAT (pkg/scopes/scopes.go:85) · [C] state change / egress: add_issue_comment, issue_write, push_files, create_pull_request, merge_pull_request (pkg/github/issues.go:1369, pkg/github/repositories.go:1639, pkg/github/pullrequests.go:1530) · Same default session? Yes

## Highest-impact improvements
1. Record every tool call (tool, arguments, result status, timestamp, authenticated user) as a structured log line by default, using the logger already injected into handlers. — C9 S L0→L2, +0.150 before caps (Playbook 1, step 3)
2. Request read-only scopes in the default OAuth login and elevate to 'repo' only through the existing per-tool scope challenge when a write tool is first used. — C1 D L1→L3, +0.100 before caps (Playbook 4)
3. Turn the ifc_labels feature on by default so every result carries an untrusted/private label the host can act on. — C5 S L2→L3, +0.075 before caps (Playbook 1, step 2)
4. Add preview/confirmation (as delete_repository already has) for merge_pull_request, delete_file and push_files, and make MCP App forms an enforced step. — C2 S L2→L3, +0.075 before caps (Playbook 5, step 1)
5. Default to the read-only tool set and require an explicit flag to enable write toolsets. — C3 D L2→L3, +0.050 before caps (Playbook 3, step 1)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was built, installed or run.
- Scored the local stdio server from the official Docker image. The README leads with the GitHub-hosted remote server (api.githubcopilot.com), whose deployment and remote-only toolsets are not in this repository and were not scored.
- HTTP mode forwards the client's bearer token to GitHub (token passthrough); C1-PASSTHRU would apply to that mode.
- Running the binary directly (not via Docker) leaves configuration loading not integrity-protected, so C6 would not be structurally absent for that deployment.
- Behaviour of the MCP go-sdk (whether raw AddTool validates input against the schema, cancellation on notifications/cancelled) and of go-github path escaping was not inspected; no rating depends on it above L2.
- Whether the baked-in OAuth client issues expiring tokens could not be determined from source.
- No text aimed at AI reviewers was found in the repository.
