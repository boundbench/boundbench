# Defense-in-depth score: Azure DevOps MCP Server (local)

**Repo:** https://github.com/microsoft/azure-devops-mcp · **Commit:** `2ffa682e2394e961e804a2739d0de62651213365` (2.10.0) · **Reviewed:** 2026-10-05
**What it is:** Microsoft's local stdio MCP server that exposes Azure DevOps boards, repos, pull requests, pipelines, wikis, test plans and search as tools.
**Category:** Coding
**Scored configuration:** Local stdio server started as the README shows (npx -y @azure-devops/mcp <org>) with no other flags: all domains loaded (every read and write tool), interactive Microsoft Entra sign-in as the user, LOG_LEVEL unset.
**Agent surface (default):** code execution no · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 5.2 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | none | **0.05** | High |
| C2 | Approval gates | L2 | L3 | L3 | L2 | 0.62 | none | **0.62** | High |
| C3 | Tool & action scoping | L2 | L3 | L2 | L1 | 0.53 | none | **0.53** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L2 | L3 | L3 | L0 | 0.53 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L2 | L2 | L2 | 0.42 | none | **0.42** | High |
| C9 | Audit & traceability | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | none | **0.33** | High |

Controls where a risk surface exists: 2.20 / 7.0 (31%); 3 criteria scored SA (surface absent).

Microsoft's local Azure DevOps MCP server signs in as you and, by default, loads every tool group, so an AI assistant can commit code, run pipelines, approve pull requests and edit work items and wikis with your full Azure DevOps rights and no confirmation step of its own. The dominant risk is prompt injection through work items, pull request comments, wiki pages or files written by others: the server labels all returned content as untrusted, but that label is only a hint, and there is no read-only mode to switch off the write side. It runs no code, keeps no memory, loads no plugins and keeps the token away from the model, but it records nothing about the actions it takes.

## Critical gaps
- In the default configuration a session that reads other people's work items, pull requests and wiki pages also holds the user's full Azure DevOps access and unattended tools that commit files, run pipelines and approve pull requests, so a prompt injection can leak private data and take irreversible actions with no human step in the server. (ASI01, ASI02, LLM01; C5). Evidence: [src/index.ts:44](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/index.ts#L44); [src/oauth.ts:8](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/oauth.ts#L8); [src/tools/pipelines.ts:52](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/pipelines.ts#L52)

## Criterion details

### C1 Identity & least privilege: 0.05 (high confidence)

By default the server signs the user in through Microsoft Entra and asks for a token with the user's full Azure DevOps permissions, then uses that one token for every tool, reads and writes alike. Tokens are short-lived and held in memory, and every request is built against the single organization named on the command line, but nothing narrows what the token itself may do. A personal access token mode lets the operator supply a token with chosen scopes, but the code does not ask for or check any scope, so least privilege depends entirely on manual setup.

- **S L0:** The default interactive sign-in requests the Azure DevOps resource's .default scope, i.e. the user's full delegated Azure DevOps authority, and the Codespaces default uses the Azure CLI developer credential chain. Evidence: [src/oauth.ts:8](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/oauth.ts#L8); [src/index.ts:25](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/index.ts#L25); [src/auth.ts:75](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/auth.ts#L75) (verified)
  - *To reach the next level:* No dedicated or narrowed identity: the server never requests per-capability scopes or separates read and write credentials.
- **C L0:** Every tool takes its client or token from one shared provider bound to the configured organization URL, but no tool call passes an authorization check; each simply uses the user's full token. Evidence: [src/index.ts:62](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/index.ts#L62); [src/index.ts:120](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/index.ts#L120) (verified)
  - *To reach the next level:* No authorization check in code on even the main tool path.
- **D L0:** The default install acts with whatever the signed-in user can do in the organization; a narrower identity requires the operator to create a scoped personal access token and switch to PAT mode. Evidence: [src/index.ts:46-52](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/index.ts#L46-L52); [src/auth.ts:41-53](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/auth.ts#L41-L53) (verified)
  - *To reach the next level:* No near-minimal default: write authority is present from the first sign-in without any explicit elevation.
- **B L1:** A hijacked session can write across the organization's repositories, pull requests, pipelines, boards and wikis with the user's rights, though only in the one configured organization and with tokens that expire within about an hour and are never written to disk by the server. Evidence: [src/index.ts:44](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/index.ts#L44); [src/tools/pipelines.ts:52](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/pipelines.ts#L52); [src/tools/repositories.ts:741](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/repositories.ts#L741) (verified)
  - *To reach the next level:* Authority is not limited to one project or to non-destructive operations; pipeline runs reach whatever service connections the pipelines hold.
- **Cap:** none
- **Notes:** PAT mode (operator-chosen PAT scopes) was not credited as an alternative mechanism: the server neither requests nor verifies scopes, so any narrowing is done outside the project.

### C2 Approval gates: 0.62 (high confidence)

The host application, not the server, decides which calls need a human, so this criterion rates the signals the server gives it. Reads and writes are separate tools, every tool carries read-only or destructive hints from one central table, and registration fails if a tool has no entry, so a new tool cannot ship unlabelled. There is no server-side confirmation step, no read-only mode, and a dry run exists only for previewing a pipeline's YAML. The server offers no delete tools, and most of its writes keep history in Azure DevOps, but pipeline runs, pull request approvals and auto-complete merges cannot be taken back.

- **S L2:** Separate read and write tools with readOnlyHint/destructiveHint from a central table that matches each tool's actions (for example pipelines_write and repo_file_write are mutating, wit_work_item is read-only). Evidence: [src/shared/tool-annotations.ts:38](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/shared/tool-annotations.ts#L38); [src/shared/tool-annotations.ts:42](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/shared/tool-annotations.ts#L42); [src/shared/tool-annotations.ts:62](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/shared/tool-annotations.ts#L62); [src/tools/pipelines.dto.ts:61](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/pipelines.dto.ts#L61) (verified)
  - *To reach the next level:* No preview or dry-run for destructive operations other than the pipeline previewRun option, and no server-enforced read-only mode or confirmation.
- **C L3:** All tool registration runs inside a proxy that injects the annotation for each tool and throws for any tool missing from the table, so every tool path carries a risk signal. Evidence: [src/shared/tool-annotations.ts:76-84](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/shared/tool-annotations.ts#L76-L84); [src/tools.ts:20-38](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools.ts#L20-L38) (verified)
  - *To reach the next level:* Unknown or unlabelled tools are rejected, but there is no server-side gate the host must complete for any mutating call.
- **D L3:** Annotations are static in code; no flag, environment variable, tool input or workspace file changes them. Evidence: [src/shared/tool-annotations.ts:26](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/shared/tool-annotations.ts#L26); searched `rg -n -S -e 'readOnlyHint|destructiveHint'` in `src` → 5 hits (all hits are the three annotation constant definitions in tool-annotations.ts; nothing else sets or overrides hints) (verified)
  - *To reach the next level:* No time-bounded elevated mode or authenticated approval exists because the server has no approval step of its own.
- **B L2:** No tool deletes work items, repositories, branches or wiki pages directly, and most writes (work items, wiki pages, commits, comments) keep revision history, but running pipelines, approving pull requests and auto-complete merges (optionally deleting the source branch) are irreversible. Evidence: [src/tools/pipelines.ts:52](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/pipelines.ts#L52); [src/tools/repositories.ts:869](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/repositories.ts#L869); searched `rg -n -S -e 'delete[A-Z]|\.delete\(|"delete"'` in `src/tools` → 5 hits (hits are the auto-complete deleteSourceBranch option and removal of pull request labels and reviewers; no tool deletes work items, repositories, branches or pages directly) (verified)
  - *To reach the next level:* No preview or rollback for external actions and no rate or quantity limits on writes.
- **Cap:** none

### C3 Tool & action scoping: 0.53 (high confidence)

The tools are purpose-built Azure DevOps operations with typed schemas and fixed action lists rather than a generic HTTP passthrough, and every request URL is built from the single organization given at startup; a wiki URL supplied by the model is refused if it points at another organization. Local file saves must be relative paths without traversal, under the server's working directory, and downloads have size caps. Some tools stay broad, though: ad-hoc WIQL queries, pipeline runs with caller-supplied variables and parameters, commits to any existing branch, and pull request auto-complete with an option to bypass branch policies. Many list sizes have no upper bound, and the default loads every tool group, writes included.

- **S L2:** Inputs are typed with enums and some length and depth bounds, requests are pinned to the configured organization URL, model-supplied wiki URLs are checked against it, and attachment saves use resolved-path containment with exclusive no-follow opens. Evidence: [src/index.ts:62](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/index.ts#L62); [src/tools/wiki.ts:178-180](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/wiki.ts#L178-L180); [src/tools/work-items.ts:526-535](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/work-items.ts#L526-L535); [src/tools/work-items.ts:332](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/work-items.ts#L332) (verified)
  - *To reach the next level:* Allowlist-grade validation is not consistent: most list sizes have no numeric bound and several tools accept free-form input (ad-hoc WIQL, pipeline variables and template parameters, policy-bypass auto-complete).
- **C L3:** Every tool has a zod schema enforced by the MCP SDK and every tool reaches Azure DevOps through the same organization-bound connection; the server has no extension tools. Evidence: [src/tools/repositories.ts:189](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/repositories.ts#L189); [src/tools/pipelines.ts:110-111](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/pipelines.ts#L110-L111) (verified)
  - *To reach the next level:* Bounds are applied per tool rather than by one central policy: most top parameters have no maximum.
- **D L2:** Tool groups (domains) are selectable with -d, but the default 'all' loads every group including repository, pipeline, wiki and work item write tools. Evidence: [src/index.ts:39-45](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/index.ts#L39-L45); [README.md:158](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/README.md#L158) (verified)
  - *To reach the next level:* The default tool set is not read-only; write tools load without explicit enabling.
- **B L1:** A misused tool can commit to any branch, run any pipeline and edit any work item or wiki the user can reach across the organization, and batch work item updates take an unbounded list. Evidence: [src/tools/repositories.ts:741](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/repositories.ts#L741); [src/tools/work-items.ts:784-785](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/work-items.ts#L784-L785) (verified)
  - *To reach the next level:* Writes are not scoped below the organization or bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation: 1.00 (high confidence)

The server never runs model-supplied code or commands: it starts no subprocesses and has no eval or template execution. The only process it opens is the system browser for the sign-in page. Pipeline runs it can trigger execute on Azure DevOps agents, not in the server, and are scored as consequential actions under approval gates and untrusted input.

- **Structural absence:** searched `rg -n -S -e 'child_process|execSync|execFile|spawn|eval\(|new Function|vm\.run'` in `src` → 0 hits (no subprocess, eval or VM execution anywhere in the server source); [src/oauth.ts:89-92](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/oauth.ts#L89-L92)

### C5 Untrusted input blast radius: 0.25 (high confidence)

Everything the server returns from Azure DevOps (work items, comments, pull request threads, repository files, wiki pages, build logs, search results) is written by other people, and every tool's text output is wrapped in randomized delimiters labelled as untrusted with the source area. That marking is consistent and cannot be switched off, but it is a hint to the model, not a limit: the same default session holds the user's full access to private code and boards and can commit, run pipelines, approve pull requests and post content with nothing in the server requiring a human. There is no read-only mode that would drop the write side.

- **S L2:** Tool outputs are serialized API objects wrapped with a per-response random nonce and an 'UNTRUSTED <source> CONTENT' label, which marks returned content apart from the server's own text. Evidence: [src/shared/content-safety.ts:21-24](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/shared/content-safety.ts#L21-L24); [src/shared/content-safety.ts:45-56](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/shared/content-safety.ts#L45-L56) (verified)
  - *To reach the next level:* No machine-readable provenance or untrusted flag the host can act on (the label is inline text), and no read-only or no-egress mode.
- **C L3:** The wrapper is applied centrally to every tool registered in every Azure DevOps domain, covering text and text-resource outputs. Evidence: [src/tools.ts:24-28](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools.ts#L24-L28); [src/tools.ts:57-62](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools.ts#L57-L62) (verified)
  - *To reach the next level:* No distinction between principals: content from any organization member is returned on the same footing.
- **D L3:** The wrapping is unconditional in code; there is no flag or environment variable to disable it and tool input cannot change it. Evidence: [src/tools.ts:24-28](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools.ts#L24-L28); searched `rg -n -S -e 'spotlight|SPOTLIGHT'` in `src/index.ts src/utils.ts src/logger.ts` → 0 hits (no option or environment switch for the content-safety wrapper) (verified)
  - *To reach the next level:* S is L2, so D is capped at L3; nothing exists beyond the always-on marking.
- **B L0:** In the default configuration a session that reads other people's work items, pull requests or wiki pages also holds the user's full organization access and unattended tools that commit files, run pipelines, approve pull requests and write wiki pages and comments, so a hijack can both leak private data into shared content and take irreversible actions. Evidence: [src/index.ts:44](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/index.ts#L44); [src/oauth.ts:8](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/oauth.ts#L8); [src/tools/pipelines.ts:52](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/pipelines.ts#L52); [src/tools/repositories.ts:869](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/repositories.ts#L869) (verified)
  - *To reach the next level:* No default mode removes a Rule-of-Two leg (read-only or no-write), and no server-side human step guards writes after untrusted content is read.
- **Cap:** C5-WORSTCASE: B is L0: in the default configuration a hijacked session can leak private data and take irreversible actions with no human step in the server.

### C6 Memory, context & configuration integrity: 1.00 (high confidence)

The server keeps no memory between sessions and loads no instruction or configuration files from the workspace: settings come from command-line flags and environment variables set by the operator. The only file it persists is a cache in the home directory mapping the organization name to its Microsoft Entra tenant, filled from an Azure DevOps response header and never shown to the model. Content the agent writes into Azure DevOps (wiki pages, work items) can of course be read back later; that exposure is scored under untrusted input.

- **Structural absence:** searched `rg -n -S -e 'dotenv|AGENTS\.md|CLAUDE\.md|memory|remember|vector|\.cursorrules'` in `src` → 0 hits (no memory store, retrieval index, dotenv loading or instruction-file loading); searched `rg -n -S -e 'readFile|writeFile'` in `src` → 4 hits (three hits are the home-directory tenant cache in org-tenants.ts (organization comes from the CLI argument); one is the attachment save, which writes downloaded content and never reads it back into context)

### C7 Third-party extensions: 1.00 (high confidence)

The server loads no third-party code at runtime: no plugins, no MCP servers of its own, no downloaded tools or models. The one dynamic import is Microsoft's own sign-in broker library, a declared package dependency.

- **Structural absence:** searched `rg -n -S -e 'plugin|import\(|npx|install'` in `src` → 7 hits (hits are the PAT fetch-interceptor installer and the dynamic import of the @azure/msal-node-extensions broker (a package dependency); no plugin loading or runtime installs)

### C8 Secrets & sensitive-data protection: 0.42 (high confidence)

The server never hands its token to the model: no tool returns it, it is not written to logs, and the default sign-in keeps short-lived tokens only in memory. In personal access token mode the server also checks the destination host before sending the token. There is, however, no masking or redaction layer at all: API error bodies are passed back to the model verbatim, and protection rests on the code simply not printing credentials. There is no telemetry, and logs go to standard error at info level by default. A leaked default token is short-lived but carries the user's full Azure DevOps rights.

- **S L1:** Credentials come from an in-memory MSAL session or environment variables and PAT mode adds a host allowlist before the token is sent, but there is no type-level masking or redaction filter anywhere. Evidence: [src/index.ts:113](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/index.ts#L113); [src/auth.ts:45](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/auth.ts#L45); searched `rg -n -S -e 'redact|mask|sanitiz|keychain|keyring'` in `src` → 1 hits (the one hit is a search-API 'sanitize' flag for result highlighting, not secret redaction) (verified)
  - *To reach the next level:* No masking or redaction for logs, error text or tool output, and no keychain or secret-manager storage for the PAT modes.
- **C L2:** By construction the token stays out of logs, tool results and model context, and there is no subprocess or telemetry path; Azure DevOps error bodies are forwarded to the model unfiltered. Evidence: searched `rg -n -e 'logger\.\w+\(.*\$\{(token|accessToken|b64Pat|basicValue|result\.token)\}'` in `src` → 0 hits (no log statement interpolates a credential); [src/tools/wiki.ts:139-142](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/wiki.ts#L139-L142) (verified)
  - *To reach the next level:* Error messages and returned content are not scanned or filtered on any path.
- **D L2:** No telemetry exists, and logging defaults to info level on standard error. Evidence: [src/logger.ts:30](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/logger.ts#L30); searched `rg -n -i -e 'telemetry|appinsights|sentry|opentelemetry'` in `src package.json` → 0 hits (no telemetry SDK or reporter) (verified)
  - *To reach the next level:* No redaction layer that is always on; capped one level above S.
- **B L2:** The default credential is a user access token that expires within about an hour and is never reachable by the model, but it carries the user's full Azure DevOps scope; PAT modes use long-lived tokens. Evidence: [src/oauth.ts:8](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/oauth.ts#L8); [src/oauth.ts:87](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/oauth.ts#L87) (verified)
  - *To reach the next level:* Tokens are not scoped to the task; the default requests the full .default scope.
- **Cap:** none

### C9 Audit & traceability: 0.00 (high confidence)

The server keeps no record of the tools it runs. Its logger writes startup and sign-in events to standard error, but no tool logs its calls, arguments or results, and there is no audit file, trace export or correlation ID. After an incident, what the agent did can only be pieced together from the host's transcript or Azure DevOps's own history, which shows the user as the actor.

- **S L0:** Tool calls are not recorded at any log level. Evidence: searched `rg -n -S 'logger\.'` in `src/tools src/tools.ts` → 0 hits (no tool handler or the registration wrapper writes a log line) (verified)
  - *To reach the next level:* No structured record of tool calls with arguments, result status and timestamps.
- **C L0:** Nothing about tool activity is recorded. Evidence: [src/index.ts:83-92](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/index.ts#L83-L92) (verified)
  - *To reach the next level:* No tool path is recorded.
- **D L0:** There is no tool-call record to turn on. Evidence: [src/logger.ts:28-38](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/logger.ts#L28-L38) (verified)
  - *To reach the next level:* No audit record exists, on by default or otherwise.
- **B L0:** With no record written, actions always proceed and nothing survives a crash. Evidence: [src/logger.ts:28-38](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/logger.ts#L28-L38) (verified)
  - *To reach the next level:* No durable per-action record.
- **Cap:** none

### C10 Limits & kill switch: 0.33 (high confidence)

A few operations have hard limits in the server: build artifact downloads are capped (1 GB to disk, 10 MB inline), ad-hoc WIQL queries are limited in length, directory listing depth is clamped, and one retry loop is bounded. Everything else is unbounded: most list sizes are whatever the model asks for, work item attachment downloads have no size cap, no request has a timeout, there is no rate limit on writes, and a cancelled call keeps running. Azure DevOps's own throttling is the main external ceiling.

- **S L2:** Server-enforced caps on some operations: artifact size limits, WIQL length, listing depth and a bounded retry loop. Evidence: [src/tools/pipelines.ts:23-24](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/pipelines.ts#L23-L24); [src/tools/repositories.ts:659](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/repositories.ts#L659); [src/tools/work-items.ts:332](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/work-items.ts#L332) (verified)
  - *To reach the next level:* No caps on every operation and no concurrency or rate limits.
- **C L1:** Limits apply only inside the individual tools that define them; there are no timeouts that would bound every call. Evidence: searched `rg -n -i -e 'timeout|abortsignal|abortcontroller|ratelimit|rate_limit|throttl'` in `src` → 1 hits (the single hit is the setTimeout backoff delay in the test-suite retry loop; no request timeouts, cancellation or rate limiting) (verified)
  - *To reach the next level:* No per-request timeout or shared limit covering every tool call.
- **D L1:** Defaults exist for page sizes, but the model can raise most of them without a ceiling. Evidence: [src/tools/pipelines.ts:171](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/pipelines.ts#L171); [src/tools/repositories.ts:189](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/repositories.ts#L189) (verified)
  - *To reach the next level:* Defaults are not bounded by maxima the model cannot exceed.
- **B L1:** A runaway caller can issue unlimited writes and pipeline runs, limited only by Azure DevOps throttling, and in-flight requests and triggered pipeline runs continue after the host stops. Evidence: [src/tools/pipelines.ts:52](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/pipelines.ts#L52); [src/tools/work-items.ts:515-520](https://github.com/microsoft/azure-devops-mcp/blob/2ffa682e2394e961e804a2739d0de62651213365/src/tools/work-items.ts#L515-L520) (verified)
  - *To reach the next level:* No server-side ceiling on writes and no cancellation of in-flight work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Work items, comments, PR threads, repo files, wiki pages and build logs written by other organization members, returned by every read tool (src/tools.ts:61) · [B] sensitive data/systems: The user's full delegated Azure DevOps access to private repositories, boards and pipelines (src/oauth.ts:8) · [C] state change / egress: Commits, pipeline runs, PR votes and auto-complete, wiki and work item writes (src/tools/repositories.ts:741, src/tools/pipelines.ts:52) · Same default session? Yes

## Highest-impact improvements
1. Add a server-enforced read-only mode (or a default domain set without write tools) so sessions that only read untrusted content cannot commit, run pipelines or approve pull requests. (C3 D L2→L3, +0.050 before caps; Playbook 3)
2. Log every tool call as a structured record (tool, arguments, result status, timestamp, signed-in user) to a file or OpenTelemetry sink by default. (C9 S L0→L2, +0.150 before caps; Playbook 1 step 3)
3. Require an MCP elicitation confirmation showing the exact call before irreversible actions (pipeline runs, PR approval and auto-complete, policy bypass). (C2 S L2→L4, +0.150 before caps; Playbook 5)
4. Request narrower delegated scopes per enabled domain (for example read-only scopes when only read tools are loaded) instead of the full .default scope. (C1 S L0→L2, +0.150 before caps; Playbook 4)
5. Add request timeouts, honour MCP cancellation, and put maximums on list sizes, attachment downloads and batch updates. (C10 C L1→L2, +0.075 before caps; Playbook 3 step 3)

## Re-audit log
- C1 C: L1 → L0. Matched the anchor literally: L1 requires an authorization check on the main tool path; the shared org-bound provider performs none.
- C3 S: L3 → L2. L3 requires allowlist validation with numeric bounds; most top parameters are unbounded and several tools take free-form queries or pipeline variables, so validation is typed but not allowlist-grade throughout.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the local stdio server as the README configures it. The README recommends Microsoft's hosted Remote MCP Server, which is not in this repository and was not scored.
- Behaviour of dependencies (azure-devops-node-api, MSAL, @azure/identity) such as default HTTP timeouts and token lifetimes was taken from their documented behaviour, not read at a pinned version.
- Azure DevOps server-side permissions and branch policies, which limit what the user's token can do, were not examined.
