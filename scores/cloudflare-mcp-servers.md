# Defense-in-Depth Score: Cloudflare MCP Servers

**Repo:** https://github.com/cloudflare/mcp-server-cloudflare · **Commit:** `ab883e51663df955316ec3191afb5b718bf4b54c` · **Reviewed:** 2026-10-03
**What it is:** Source for Cloudflare's MCP servers (Workers, DNS analytics, observability, etc.)
**Category:** Infrastructure & Ops
**Scored configuration:** The hosted remote MCP servers as configured for the production environment in each apps/*/wrangler.jsonc, with OAuth by default and direct Cloudflare API-token auth also accepted.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents no · external communication no

## Score: 3.9 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L3 | L2 | L1 | 0.53 | C1-PASSTHRU | **0.25** | High |
| C2 | Approval gates | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C4 | Code-execution isolation | L2 | L3 | L2 | L2 | 0.57 | — | **0.57** | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L2 | 0.10 | C7-RCELOAD | **0.10** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C9 | Audit & traceability | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C10 | Limits & kill switch | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |

Controls where a risk surface exists: 3.03 / 9.0 (34%); 1 criterion scored SA (surface absent).

These servers act on your Cloudflare account with your own OAuth grant or API token, and each server asks only for the scopes its domain needs. The main risk is that the server gives your MCP client little help in deciding what needs approval: the bindings server can irreversibly delete databases and buckets, the D1 SQL tool is labelled non-destructive, and the container and remote-capture tools carry no risk labels. Every server also accepts and forwards raw API tokens, and the container server runs model-chosen shell commands and packages as root with open internet and no timeout.

## Critical gaps
- Every authenticated server accepts a client-supplied Cloudflare API token and forwards it unchanged to the Cloudflare API (token passthrough). (ASI03, T3, T9; C1) — [packages/mcp-common/src/oauth-router.ts:108-111](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-common/src/oauth-router.ts#L108-L111); [packages/mcp-common/src/api-token-mode.ts:99-104](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-common/src/api-token-mode.ts#L99-L104)
- The container server has the model install and run unverified pip/npm packages, as root with full internet, without user consent. (ASI04, T17, LLM03; C7) — [apps/sandbox-container/server/prompts.ts:16-20](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/server/prompts.ts#L16-L20); [apps/sandbox-container/server/containerHelpers.ts:25-27](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/server/containerHelpers.ts#L25-L27)

## Criterion details

### C1 Identity & least privilege — 0.25 (high)

Each server asks Cloudflare's OAuth for its own scope list (the observability server only asks for read scopes; the bindings server asks for Workers and D1 write), and every account-scoped tool checks that the chosen account is one the user's credential can actually access. Read and write tools on the same server share one credential, and the scopes always include offline_access, so the server stores a 30-day refresh token. Every server also accepts a raw Cloudflare API token from the client and forwards it unchanged to the Cloudflare API (token passthrough), so the server's authority is whatever token the client hands it. That passthrough caps this criterion.

- **S L2:** Per-server OAuth scope lists are role-scoped, but read and write tools on a server share one token, and API-token mode forwards whatever scope the client token carries. — [apps/workers-bindings/src/bindings.app.ts:14-20](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/workers-bindings/src/bindings.app.ts#L14-L20); [packages/mcp-common/src/cloudflare-oauth-handler.ts:494](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-common/src/cloudflare-oauth-handler.ts#L494); [packages/mcp-common/src/oauth-router.ts:108-111](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-common/src/oauth-router.ts#L108-L111); [packages/mcp-common/src/api-token-mode.ts:99-104](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-common/src/api-token-mode.ts#L99-L104) (verified)
  - *To reach the next level:* Read and write tools on the same server do not use separate, narrower credentials, and passthrough tokens are not narrowed at all.
- **C L3:** All tools reach Cloudflare through the per-request user token, and account-scoped tools validate the header/argument account against the token's own account list, failing closed when props are missing. — [packages/mcp-common/src/account-manager.ts:99-108](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-common/src/account-manager.ts#L99-L108); [packages/mcp-common/src/server.ts:188-195](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-common/src/server.ts#L188-L195) (verified)
  - *To reach the next level:* Authorization is Cloudflare's own check on whatever token arrives; there is no server-side policy intersecting the requesting user with a least-privilege policy per tool.
- **D L2:** OAuth is the default and requests a per-server scope set, but the bindings server requests write scopes by default and any client can widen authority by sending a broader API token. — [apps/workers-bindings/src/bindings.app.ts:17-19](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/workers-bindings/src/bindings.app.ts#L17-L19); [server.json:19-26](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/server.json#L19-L26) (verified)
  - *To reach the next level:* Default should be read-only scopes with write requiring an explicit elevation, and client-supplied tokens should not widen authority.
- **B L1:** A hijacked bindings-server session can create and delete D1 databases, R2 buckets, KV namespaces and Hyperdrive configs across every account the user belongs to; a passthrough account token can be broader still. — [apps/workers-bindings/src/tools/d1.tools.ts:128](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/workers-bindings/src/tools/d1.tools.ts#L128); [packages/mcp-common/src/oauth-router.ts:101-102](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-common/src/oauth-router.ts#L101-L102); [packages/mcp-common/src/scopes.ts:4](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-common/src/scopes.ts#L4) (verified)
  - *To reach the next level:* Writes are not limited to one account or to non-destructive operations, and refresh tokens live 30 days.
- **Cap:** C1-PASSTHRU — resolveExternalToken accepts a client-supplied Cloudflare API token and the tools forward it unchanged to api.cloudflare.com.

### C2 Approval gates — 0.20 (high)

These are tool servers, so the approval prompt belongs to the MCP client; the server's job is to label which tools are risky. Only some servers set read-only/destructive hints. The container server (shell execution, file delete), the DEX server (remote packet captures on employees' devices), the Browser Run server and the URL scanner carry no hints, and the D1 query tool, which runs arbitrary SQL including DROP/DELETE, is explicitly labelled non-destructive. There is no dry-run, no read-only mode and no server-side confirmation; the DEX capture tools ask the model in their description to confirm with the user, which is not a control. Deleting databases, buckets and namespaces is irreversible.

- **S L1:** Annotations exist on bindings, builds and shared tools but are missing on many mutating tools and wrong on d1_database_query. — [apps/workers-bindings/src/tools/d1.tools.ts:193-206](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/workers-bindings/src/tools/d1.tools.ts#L193-L206); searched `rg -n readOnlyHint|destructiveHint` in `apps/sandbox-container/server apps/dex-analysis/src apps/browser-rendering/src apps/radar/src apps/autorag/src apps/ai-gateway/src apps/graphql/src apps/logpush/src` → 0 hits (No annotations at all on the container, DEX, Browser Run, Radar/URL-scanner, AutoRAG, AI Gateway, GraphQL or Logpush tools.); [apps/sandbox-container/server/container-tools.ts:60-66](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/server/container-tools.ts#L60-L66) (verified)
  - *To reach the next level:* Every tool needs accurate readOnlyHint/destructiveHint, with reads and writes in separate tools.
- **C L1:** Mutating tools such as container_exec, container_file_delete, dex_create_remote_pcap, kill_browser_session and create_url_scan carry no risk signal from the server. — [apps/dex-analysis/src/tools/dex-analysis.tools.ts:200-203](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/dex-analysis/src/tools/dex-analysis.tools.ts#L200-L203); [apps/browser-rendering/src/tools/browser.tools.ts:611](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/browser-rendering/src/tools/browser.tools.ts#L611) (verified)
  - *To reach the next level:* All mutating tools would need flags so the host's gate applies uniformly.
- **D L1:** Risk signalling is static in code and cannot be changed by the model, but the server offers no enforced confirmation or read-only mode, so any gate depends entirely on the host; the DEX tools rely on a prompt instruction to confirm. — [apps/dex-analysis/src/tools/dex-analysis.tools.ts:203](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/dex-analysis/src/tools/dex-analysis.tools.ts#L203) (verified)
  - *To reach the next level:* A server-enforced read-only mode or confirmation step would be needed.
- **B L0:** A wrongly approved or unflagged call can irreversibly delete D1 databases, R2 buckets, KV namespaces and Hyperdrive configs, or start packet captures on real user devices, with no undo or preview. — [apps/workers-bindings/src/tools/d1.tools.ts:113-128](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/workers-bindings/src/tools/d1.tools.ts#L113-L128); [apps/dex-analysis/src/tools/dex-analysis.tools.ts:236-251](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/dex-analysis/src/tools/dex-analysis.tools.ts#L236-L251) (verified)
  - *To reach the next level:* Destructive operations need previews/dry-runs or recovery, and quantities need bounds.
- **Cap:** none
- **Notes:** C2-POWERBYPASS was considered for d1_database_query (arbitrary SQL labelled non-destructive) but not applied because the gate is host-owned; the mislabel is reflected in S.

### C3 Tool & action scoping — 0.45 (high)

Tool inputs are typed with zod schemas and the account ID is checked against an allowlist of the user's accounts, but many tools are general-purpose: arbitrary shell in the container, arbitrary SQL against D1, arbitrary GraphQL, any URL for Browser Run, and a Hyperdrive edit that can point a database connection at any host. Several free-form IDs (crawl job and browser session IDs) are pasted into API paths without validation. Each server ships its full tool set, so the bindings server exposes create/delete tools by default. A misused tool acts on production resources in the user's Cloudflare accounts.

- **S L2:** Typed schemas and an account allowlist exist, but SQL, shell, URL and host arguments are passed through and some IDs are interpolated into API paths unvalidated. — [apps/workers-bindings/src/tools/d1.tools.ts:196-200](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/workers-bindings/src/tools/d1.tools.ts#L196-L200); [apps/workers-bindings/src/tools/hyperdrive.tools.ts:190](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/workers-bindings/src/tools/hyperdrive.tools.ts#L190); [apps/browser-rendering/src/tools/browser.tools.ts:505-508](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/browser-rendering/src/tools/browser.tools.ts#L505-L508) (verified)
  - *To reach the next level:* Arguments would need allowlist validation (path-safe IDs, host allowlists, bounded or parameterized queries).
- **C L2:** Most tools have schemas, and account resolution is shared through accountTool, but argument validation beyond types is per-tool and absent on many. — [packages/mcp-common/src/registration-context.ts:128-146](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-common/src/registration-context.ts#L128-L146) (verified)
  - *To reach the next level:* A shared validation layer covering all tools' risky arguments is missing.
- **D L2:** Tools are grouped by server, and most servers are read-oriented, but the bindings and container servers ship write, delete and exec tools by default with no read-only toggle. — [apps/workers-bindings/src/bindings.app.ts:26-33](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/workers-bindings/src/bindings.app.ts#L26-L33); [apps/sandbox-container/server/container-tools.ts:60-70](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/server/container-tools.ts#L60-L70) (verified)
  - *To reach the next level:* A read-only default with explicit enabling of write/exec tools is missing.
- **B L1:** A misused tool can delete or rewrite production Cloudflare resources across every account the user can access, with no quantity bounds. — [apps/workers-bindings/src/tools/d1.tools.ts:208-216](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/workers-bindings/src/tools/d1.tools.ts#L208-L216); [apps/workers-bindings/src/tools/hyperdrive.tools.ts:251-255](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/workers-bindings/src/tools/hyperdrive.tools.ts#L251-L255) (verified)
  - *To reach the next level:* Tools would need to be scoped to one account/project and quantity-bounded.
- **Cap:** none

### C4 Code-execution isolation — 0.57 (medium)

Only the container server runs model-written commands: container_exec passes the string to a shell inside a per-user Cloudflare Container, and nothing runs on the Worker itself. The container image in the repo is a stock Alpine image running as root with full internet access, and production deploys a pinned registry image that cannot be checked against the repo. No credentials are passed into the container. Exec has no timeout (the timeout argument is accepted but ignored), and a dev/test environment setting routes calls to a local process instead of a container.

- **S L2:** Execution happens in a per-user platform container; the repo's Dockerfile is a stock Alpine image with no USER (root) and no hardening, and the production image is a pinned registry tag not built from this commit. — [apps/sandbox-container/container/sandbox.container.app.ts:138-140](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/container/sandbox.container.app.ts#L138-L140); [apps/sandbox-container/Dockerfile:2](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/Dockerfile#L2); [apps/sandbox-container/wrangler.jsonc:136](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/wrangler.jsonc#L136) (inferred)
  - *To reach the next level:* A hardened (non-root, dropped capabilities, read-only root) or microVM/gVisor-class boundary would need to be shown in code.
- **C L3:** Every model-reachable exec and file path goes through the user's container Durable Object; the only exception is the dev/test environment, documented as a local-dev hack. — [apps/sandbox-container/server/container-tools.ts:12-19](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/server/container-tools.ts#L12-L19); [apps/sandbox-container/CONTRIBUTING.md:7](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/CONTRIBUTING.md#L7) (verified)
  - *To reach the next level:* There is no fail-closed guarantee covering spawned processes, and the local fallback is driven by an environment value.
- **D L2:** The container path is always used in production, but the ENVIRONMENT value 'dev' or 'test' silently routes execution to localhost. — [apps/sandbox-container/server/containerHelpers.ts:8-11](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/server/containerHelpers.ts#L8-L11); [apps/sandbox-container/server/containerHelpers.ts:70-74](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/server/containerHelpers.ts#L70-L74) (verified)
  - *To reach the next level:* Disabling the sandbox should require an explicit, loudly named operator flag rather than an environment name.
- **B L2:** Inside the container the process runs as root with unrestricted internet (enableInternet: true), but holds no Cloudflare credentials; containers are reaped after 15 minutes only when capacity is half used. — [apps/sandbox-container/server/containerHelpers.ts:25-27](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/server/containerHelpers.ts#L25-L27); [apps/sandbox-container/server/containerManager.ts:40-41](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/server/containerManager.ts#L40-L41) (verified)
  - *To reach the next level:* Network egress would need to be off or allowlisted, with CPU/memory/time limits in code.
- **Cap:** none

### C5 Untrusted input blast radius — 0.05 (high)

Several servers return attacker-influenced content to the model: web pages from Browser Run, Workers logs (anyone who can hit the user's Worker can write log lines), AI Gateway prompt logs, D1 and KV data, and DEX diagnostic files. Results are returned as plain JSON text with no marking of what is untrusted, and the DEX server mixes its own instructions to the model into the same output as the data. The server has no read-only or no-egress mode. A hijacked session on the bindings server can irreversibly delete databases and buckets unattended; leaking data out needs a second server or a less direct path, such as repointing a Hyperdrive config.

- **S L0:** Outputs are untagged JSON text, and DEX outputs carry model directives (llmContext) alongside API data. — [apps/dex-analysis/src/tools/dex-analysis.tools.ts:608-614](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/dex-analysis/src/tools/dex-analysis.tools.ts#L608-L614); [apps/workers-bindings/src/tools/d1.tools.ts:217-223](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/workers-bindings/src/tools/d1.tools.ts#L217-L223) (verified)
  - *To reach the next level:* Outputs need structured separation of content from metadata, plus provenance flags the host can act on.
- **C L0:** No untrusted source is distinguished from trusted data in any server. — [apps/browser-rendering/src/tools/browser.tools.ts:22-48](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/browser-rendering/src/tools/browser.tools.ts#L22-L48) (verified)
  - *To reach the next level:* At least one untrusted source (e.g., fetched web pages) would need to be marked.
- **D L0:** No mechanism exists to be on by default. — searched `rg -n -i untrusted|provenance|readOnlyMode|read_only` in `apps packages/mcp-common/src -g *.ts -g !worker-configuration.d.ts -g !*.spec.ts` → 0 hits (No untrusted-content marking or read-only mode anywhere in server code.) (verified)
  - *To reach the next level:* An untrusted-content marking or read-only mode would need to exist and be on by default.
- **B L1:** With the bindings server, a hijack can irreversibly delete D1/R2/KV/Hyperdrive resources unattended; a clean exfiltration channel inside that one server is limited (Hyperdrive host repointing), though hosts routinely combine it with Browser Run. — [apps/workers-bindings/src/tools/d1.tools.ts:113-128](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/workers-bindings/src/tools/d1.tools.ts#L113-L128); [apps/workers-bindings/src/tools/hyperdrive.tools.ts:190](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/workers-bindings/src/tools/hyperdrive.tools.ts#L190) (verified)
  - *To reach the next level:* Irreversible actions would need server-side confirmation or a read-only mode so only reversible changes happen unattended.
- **Cap:** none

### C6 Memory, context & configuration integrity — 1.00 (high)

The servers keep no model-writable memory and auto-load no instruction or configuration files. Durable state is OAuth grants, an identity cache keyed by token hash, and per-user container working files that are only read when the model asks for them. Server instructions are static strings plus the user's account list from Cloudflare's API. There is nothing here for an injection to persist into.

- **Structural absence:** searched `rg -n -i remember|save_memory|load_dotenv|dotenv|AGENTS\.md|CLAUDE\.md|\.cursorrules -g *.ts -g !worker-configuration.d.ts` in `apps packages` → 0 hits (No memory tool, dotenv loading or instruction-file loading in server code (the only raw hits are generated Workers type definitions, excluded).); searched `rg -n DurableKVStore` in `apps` → 0 hits (The generic Durable Object KV helper in mcp-common is not used by any app.)

### C7 Third-party extensions — 0.10 (high)

The servers load no plugins or third-party MCP servers. The container server, however, tells the model to install whatever packages it needs, so pip/npm packages chosen by the model are downloaded and run without verification or consent. That code is confined to the user's container, which holds no Cloudflare credentials but has full internet access and runs as root.

- **S L0:** Model-chosen package installs run unverified inside the container, and the server instructions encourage them. — [apps/sandbox-container/server/prompts.ts:16-20](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/server/prompts.ts#L16-L20) (verified)
  - *To reach the next level:* Installs would need pinned versions or an allowlisted registry.
- **C L0:** No extension type is verified. — [apps/sandbox-container/container/sandbox.container.app.ts:138-140](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/container/sandbox.container.app.ts#L138-L140) (verified)
  - *To reach the next level:* At least package installs would need verification.
- **D L0:** Packages are installed whenever the model runs pip/npm, with no consent step from the server. — [apps/sandbox-container/server/prompts.ts:20](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/server/prompts.ts#L20) (verified)
  - *To reach the next level:* Installing third-party code should require explicit user consent showing the package.
- **B L2:** A malicious package runs in the user's container separate from the Worker and with no Cloudflare credentials, but as root with full internet and access to all of the user's container files. — [apps/sandbox-container/server/containerHelpers.ts:25-27](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/server/containerHelpers.ts#L25-L27) (verified)
  - *To reach the next level:* Per-extension sandboxing with its own scoped access is missing.
- **Cap:** C7-RCELOAD — By default the container server has the model download and run unverified packages (pip/npm) without user consent.

### C8 Secrets & sensitive-data protection — 0.50 (high)

Cloudflare credentials stay on the server side: the model never sees them, the identity cache stores a SHA-256 of API tokens rather than the token, and Sentry reporting uses a header allowlist that excludes Authorization and strips query parameters other than scope. Upstream OAuth tokens and 30-day refresh tokens are stored in the OAuth provider's grant props, whose encryption is library behaviour rather than code here. Errors returned to the model include upstream error text, and Workers traces are enabled by default. Leaked tokens are scoped per server but refresh tokens are long-lived, and passthrough API tokens can be broader.

- **S L2:** Header and search-param allowlists for Sentry and hashing of cached API tokens are in code; at-rest encryption of stored tokens relies on the provider library. — [packages/mcp-common/src/sentry.ts:69-80](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-common/src/sentry.ts#L69-L80); [packages/mcp-common/src/api-token-mode.ts:59](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-common/src/api-token-mode.ts#L59) (verified)
  - *To reach the next level:* Redaction before model-bound messages and verified encryption at rest are not shown in code.
- **C L2:** Telemetry (Sentry) and the identity cache are protected and tokens never enter tool outputs, but upstream error bodies flow into tool results and console logs without redaction. — [apps/workers-bindings/src/tools/d1.tools.ts:230](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/workers-bindings/src/tools/d1.tools.ts#L230) (verified)
  - *To reach the next level:* Error messages and console logs would need redaction too.
- **D L2:** Sentry and Workers traces are on by default but go to the operator's own first-party services, with request headers allowlisted; metrics are content-free. — [apps/workers-bindings/wrangler.jsonc:27-33](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/workers-bindings/wrangler.jsonc#L27-L33); [packages/mcp-observability/src/metrics.ts:20-21](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-observability/src/metrics.ts#L20-L21) (verified)
  - *To reach the next level:* Telemetry would need to be opt-in or content-free with redaction always on.
- **B L2:** Stored credentials are scoped per server but include 30-day refresh tokens, and passthrough API tokens can be long-lived and broad. — [packages/mcp-common/src/oauth-router.ts:101-102](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-common/src/oauth-router.ts#L101-L102) (verified)
  - *To reach the next level:* Credentials would need to be short-lived and rotatable end to end.
- **Cap:** none

### C9 Audit & traceability — 0.38 (high)

Every tool registered through the shared layer emits a metric to Workers Analytics Engine with the tool name, the user ID and an error code, written by server code the model cannot influence. It does not record arguments, the target account or results, so you cannot reconstruct what was deleted or executed. Tools that return an error result instead of throwing are recorded as successes, and the metric is written after the call, best-effort, with failures only logged to the console.

- **S L1:** Records hold tool name, user ID and error code only, with no arguments, account or result. — [packages/mcp-observability/src/metrics.ts:15-24](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-observability/src/metrics.ts#L15-L24) (verified)
  - *To reach the next level:* Each call's arguments, target account and result status would need to be recorded.
- **C L2:** All tools registered via registerTool/accountTool pass through trackTool. — [packages/mcp-common/src/registration-context.ts:106-118](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-common/src/registration-context.ts#L106-L118) (verified)
  - *To reach the next level:* Coverage of approvals/denials and of the target resource is missing.
- **D L2:** Metrics are on by default in every production wrangler config and written to Analytics Engine outside model control. — [apps/workers-bindings/wrangler.jsonc:118-123](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/workers-bindings/wrangler.jsonc#L118-L123) (verified)
  - *To reach the next level:* Records written by a component the model can't control are present, but the record content is too thin to rely on (level limited by Strength).
- **B L1:** The metric is written after the tool runs, failures are caught and only console-logged, and error results returned without throwing are logged as successes. — [packages/mcp-observability/src/analytics-engine.ts:18-26](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-observability/src/analytics-engine.ts#L18-L26); [packages/mcp-common/src/registration-context.ts:161-164](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-common/src/registration-context.ts#L161-L164) (verified)
  - *To reach the next level:* Records would need to be written per action with errors surfaced.
- **Cap:** none

### C10 Limits & kill switch — 0.38 (high)

The servers cap MCP request bodies at 4 MB, and the container server limits the fleet to 50 containers and reaps containers older than 15 minutes, but only when capacity is already half used. Shell commands in the container have no timeout: the timeout argument is declared but never applied, so a process can run until the container is reaped. Crawl depth and page limits are left to the model, and there is no per-user rate limit in code. Workers' platform CPU limits are not credited because they are not configured here.

- **S L2:** Server-enforced caps exist on request size and container count, but exec has no timeout and there are no rate limits. — [packages/mcp-common/src/server.ts:112](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/packages/mcp-common/src/server.ts#L112); [apps/sandbox-container/server/containerHelpers.ts:1](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/server/containerHelpers.ts#L1); [apps/sandbox-container/shared/schema.ts:6](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/shared/schema.ts#L6) (verified)
  - *To reach the next level:* Caps on every operation plus concurrency/rate limits would be needed.
- **C L1:** Bounds cover the HTTP request and container count only; container exec and crawl jobs are unbounded. — [apps/sandbox-container/container/sandbox.container.app.ts:138-141](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/container/sandbox.container.app.ts#L138-L141) (verified)
  - *To reach the next level:* Tool-level timeouts on exec and long-running jobs are missing.
- **D L2:** Limits are hard-coded constants the model cannot raise, but the crawl limit/depth and exec timeout are model-chosen or ignored. — [apps/browser-rendering/src/tools/browser.tools.ts:435-436](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/browser-rendering/src/tools/browser.tools.ts#L435-L436) (verified)
  - *To reach the next level:* Defaults for every operation that the model cannot raise are missing.
- **B L1:** A runaway exec keeps running until the container is reaped, which only happens under capacity pressure, and started crawls continue after the call returns. — [apps/sandbox-container/server/userContainer.ts:43-44](https://github.com/cloudflare/mcp-server-cloudflare/blob/ab883e51663df955316ec3191afb5b718bf4b54c/apps/sandbox-container/server/userContainer.ts#L43-L44) (verified)
  - *To reach the next level:* Tight per-call time ceilings and cancellation of in-flight work are missing.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: D1/KV contents and Workers logs written by end users of the customer's apps; web pages via Browser Run (apps/browser-rendering/src/tools/browser.tools.ts:35) · [B] sensitive data/systems: Customer D1 query results and account data (apps/workers-bindings/src/tools/d1.tools.ts:221) · [C] state change / egress: Irreversible deletes of D1/R2/KV/Hyperdrive (apps/workers-bindings/src/tools/d1.tools.ts:128) and Hyperdrive host repointing (apps/workers-bindings/src/tools/hyperdrive.tools.ts:252) · Same default session? Yes

## Highest-impact improvements
1. Add accurate readOnlyHint/destructiveHint to every tool (container, DEX, Browser Run, URL scanner) and mark d1_database_query and hyperdrive_config_edit destructive. — C2 S L1→L2, +0.075 before caps (Playbook 5)
2. Stop accepting raw Cloudflare API tokens on the MCP endpoint (or exchange them for server-issued, down-scoped tokens) so the server only acts on tokens issued for it. — C1 S L2→L3, +0.075 before caps (Playbook 4)
3. Record tool arguments (redacted), target account and true result status in the tool-call metric. — C9 S L1→L2, +0.075 before caps (Playbook 1 step 3)
4. Apply a server-side exec timeout and kill the process tree; reap containers on a fixed schedule, not only under capacity pressure. — C10 C L1→L2, +0.075 before caps (Playbook 3 step 3)
5. Run the sandbox as a non-root user with an egress allowlist and resource limits. — C4 B L2→L3, +0.050 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The whole monorepo (17 apps plus packages/mcp-common) was scored as one project; per-server risk differs widely (most servers are read-only, workers-bindings and sandbox-container carry the write/exec surface). Radar (3.4k lines) and GraphQL tool files were sampled, not read line by line.
- Behaviour of @cloudflare/workers-oauth-provider (grant-prop encryption, token handling) and of the Cloudflare Containers platform (VM isolation, resource limits) was not verified and not credited.
- The production container image is a pinned registry tag (wrangler.jsonc) that cannot be tied to the repo's Dockerfile at this commit.
- The recommended Code Mode server lives in a different repository (cloudflare/mcp) and is out of scope.
- No reviewer-injection text aimed at AI auditors was found in the repository.
