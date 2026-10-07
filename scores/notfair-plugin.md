# Defense-in-Depth Score: NotFair Plugin

**Repo:** https://github.com/nowork-studio/notfair-plugin · **Commit:** `1a53d4e4da69a6060c24de42c23333fb48e927c4` (0.27.15) · **Reviewed:** 2026-10-04
**What it is:** Host-agnostic marketing plugin: 48 SEO, GEO and paid-media skills plus one remote NotFair MCP connection for ad, analytics, CMS and CRM platforms.
**Category:** Data & Analytics
**Scored configuration:** Plugin installed per README (Claude Code / Codex marketplace) with its default .mcp.json NotFair connection and bundled local scripts; the optional notfair/ goal-loop app is not scored.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents no · external communication yes

## Score: 1.2 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C2 | Approval gates | L0 | L0 | L1 | L0 | 0.05 | C2-SELFAPPROVE | **0.05** | High |
| C3 | Tool & action scoping | L2 | L2 | L0 | L1 | 0.35 | — | **0.35** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L0 | 0.07 | C5-WORSTCASE | **0.07** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | L1 | L0 | L1 | L0 | 0.12 | — | **0.12** | High |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L1 | L0 | 0.05 | C8-MODELSECRETS | **0.05** | High |
| C9 | Audit & traceability | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L0 | 0.35 | — | **0.35** | High |


NotFair gives a coding agent one OAuth connection that can change ad budgets, campaigns, WordPress sites and CRM records across ten platforms, and almost every safeguard it ships is instruction text for the model, not code. The one write script in the repo has a confirmation step the model skips with --yes. The setup flow collects CMS API keys and passwords through chat, and the CMS scripts trust .env files from the working directory. Install it only on a host whose own approval prompts you keep on for every MCP call.

## Critical gaps
- The plugin's only code-level write confirmation is skipped by a model-supplied --yes flag, and all MCP mutations rely on prompt text for approval. (ASI09, ASI02, T10; C2) — [seo/seo-analysis/scripts/push_strapi_seo.py:300-305](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/push_strapi_seo.py#L300-L305); [seo/seo-analysis/scripts/push_strapi_seo.py:426](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/push_strapi_seo.py#L426); [paid-ads/shared/operating-contract.md:10](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/paid-ads/shared/operating-contract.md#L10)
- Bundled scripts and pre-approved Bash run unsandboxed as the user with ambient gcloud credentials and full network. (ASI05, T11; C4) — [seo/setup-cms/SKILL.md:12-13](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/setup-cms/SKILL.md#L12-L13); [notfair-upgrade-skill/SKILL.md:9-10](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/notfair-upgrade-skill/SKILL.md#L9-L10); [seo/seo-analysis/scripts/_gcloud.py:35-38](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/_gcloud.py#L35-L38)
- Sessions combine untrusted web/CMS content, sensitive ad and CRM data, and write/egress capabilities with no plugin-enforced break (C5-WORSTCASE). (ASI01, LLM01, T6; C5) — [seo/broken-link-checker/scripts/checker.py:96-112](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/broken-link-checker/scripts/checker.py#L96-L112); [paid-ads/shared/operating-contract.md:10](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/paid-ads/shared/operating-contract.md#L10); [.mcp.json:1-8](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/.mcp.json#L1-L8)
- CMS scripts auto-load .env/.env.local from the working directory and parents, letting a workspace file redirect the endpoint that receives the Bearer API key. (ASI06, T1; C6) — [seo/seo-analysis/scripts/push_strapi_seo.py:121-130](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/push_strapi_seo.py#L121-L130); [seo/seo-analysis/scripts/push_strapi_seo.py:140](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/push_strapi_seo.py#L140); [seo/seo-analysis/scripts/push_strapi_seo.py:153](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/push_strapi_seo.py#L153)
- Upgrades install unverified latest main into the plugin cache, and updated code runs as the user with full environment and OAuth reach. (ASI04, T17; C7) — [notfair-upgrade-skill/SKILL.md:66-81](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/notfair-upgrade-skill/SKILL.md#L66-L81); [notfair-upgrade-skill/SKILL.md:9-10](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/notfair-upgrade-skill/SKILL.md#L9-L10)
- setup-cms collects CMS API keys and WordPress application passwords through chat, sending them to the model provider. (ASI03, LLM02; C8) — [seo/setup-cms/SKILL.md:86-95](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/setup-cms/SKILL.md#L86-L95); [seo/setup-cms/SKILL.md:119](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/setup-cms/SKILL.md#L119)

## Criterion details

### C1 Identity & least privilege — 0.05 (high)

The plugin itself holds no identity of its own: it points the host at one NotFair OAuth connection that, once authorized, can read and write across Google, Meta, X, LinkedIn, Reddit and TikTok Ads, GA4, Search Console, WordPress and GoHighLevel. Which scopes that connection carries is decided by NotFair's private server, not by anything in this repository. The bundled Search Console scripts mint the user's ambient gcloud credentials with the broad cloud-platform scope and fall back to an unscoped token. Authorization beyond that is described only in skill prose (a saved account ID is not proof of access).

- **S L0:** Local scripts use the operator's ambient gcloud Application Default Credentials with the cloud-platform scope, falling back to an unscoped token; the MCP connection's scopes are not defined anywhere in the repo. — [seo/seo-analysis/scripts/_gcloud.py:35-38](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/_gcloud.py#L35-L38); [seo/seo-analysis/scripts/_gcloud.py:55](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/_gcloud.py#L55); [.mcp.json:1-8](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/.mcp.json#L1-L8) (verified)
  - *To reach the next level:* No dedicated, role-scoped identity: the scripts request cloud-platform rather than only webmasters.readonly, and read and write MCP operations share one OAuth grant.
- **C L0:** There is no authorization layer in plugin code; account/target authorization lives only in skill prompts. — [google-ads/shared/preamble.md:13-15](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/google-ads/shared/preamble.md#L13-L15); searched `rg -n -i 'authoriz|permission|scope'` in `seo/seo-analysis/scripts/_gcloud.py seo/seo-analysis/scripts/push_strapi_seo.py` → 11 hits (Hits are the GSC OAuth scope constant and its comments, a PermissionError handler, and the Bearer header; none is an authorization check.) (verified)
  - *To reach the next level:* No code path checks a request against a scoped identity before credentials are used.
- **D L0:** The default install registers one connection whose manifest advertises Write capability across all platforms; no read-only default exists. — [.codex-plugin/plugin.json:27-30](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/.codex-plugin/plugin.json#L27-L30); [.mcp.json:1-8](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/.mcp.json#L1-L8) (verified)
  - *To reach the next level:* No read-only default connection; write is not a separate operator elevation.
- **B L1:** A hijacked session holds write access to ad spend, CMS/WordPress, and CRM systems across several vendors plus the user's gcloud cloud-platform token. — [seo/seo-analysis/scripts/_gcloud.py:35-38](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/_gcloud.py#L35-L38); [docs/mcp-connection.md:3-7](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/docs/mcp-connection.md#L3-L7) (verified)
  - *To reach the next level:* Blast radius spans multiple write-capable systems; it is not confined to one system or to read access.
- **Cap:** none

### C2 Approval gates — 0.05 (high)

Every consequential action this plugin enables (budget, bid and campaign changes, CRM edits, WordPress changes, sitemap submissions) flows through NotFair's remote MCP, and the plugin's only safeguard on that path is instruction text telling the model to get explicit approval. The one write path implemented in this repo, the Strapi SEO push script, shows a real diff and asks y/N, but it refuses to prompt when not on a terminal and tells the caller to pass --yes, which the model can simply add. Two skills also pre-approve unrestricted Bash in their frontmatter, removing the host's per-command prompt while they run. Whatever the host's own approval UI does is not credited to this plugin.

- **S L0:** The Strapi push script prints a per-entry diff and asks for confirmation, but the model can skip it with --yes; no risk annotations, read/write separation or dry-run is provided for the MCP path. — [seo/seo-analysis/scripts/push_strapi_seo.py:366-369](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/push_strapi_seo.py#L366-L369); [seo/seo-analysis/scripts/push_strapi_seo.py:300-305](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/push_strapi_seo.py#L300-L305); [seo/seo-analysis/scripts/push_strapi_seo.py:426](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/push_strapi_seo.py#L426); searched `rg -n 'readOnlyHint|destructiveHint'` in `.mcp.json mcp.json .claude-plugin .codex-plugin seo/seo-analysis/scripts` → 0 hits (verified)
  - *To reach the next level:* No risk signalling (read-only/destructive hints) on any tool the plugin wires up; the only confirmation is model-skippable.
- **C L0:** The most powerful path, remote MCP mutations across ad and CRM platforms, has no plugin-side gate beyond prompt text, and two skills pre-approve Bash. — [paid-ads/shared/operating-contract.md:10](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/paid-ads/shared/operating-contract.md#L10); [seo/setup-cms/SKILL.md:12-13](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/setup-cms/SKILL.md#L12-L13); [notfair-upgrade-skill/SKILL.md:9-10](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/notfair-upgrade-skill/SKILL.md#L9-L10) (verified)
  - *To reach the next level:* MCP writes and Bash-run scripts are not covered by any code-level gate.
- **D L1:** The script confirmation is on by default but the model defeats it by passing --yes, which non-interactive execution requires. — [seo/seo-analysis/scripts/push_strapi_seo.py:300-305](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/push_strapi_seo.py#L300-L305); [seo/seo-analysis/scripts/push_strapi_seo.py:426](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/push_strapi_seo.py#L426) (verified)
  - *To reach the next level:* The model can switch the confirmation off from its own command line.
- **B L0:** Wrongly approved MCP calls can raise budgets, enable campaigns, delete entities, and change CRM or WordPress state with no plugin-side bound or rollback. — [paid-ads/shared/operating-contract.md:10](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/paid-ads/shared/operating-contract.md#L10); [.mcp.json:1-8](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/.mcp.json#L1-L8) (verified)
  - *To reach the next level:* No plugin-enforced spend or quantity bounds, previews, or rollback on external mutations.
- **Cap:** C2-SELFAPPROVE — The only code-level confirmation (push_strapi_seo.py) is satisfied by the model passing --yes.

### C3 Tool & action scoping — 0.35 (high)

The CMS scripts validate their configured base URL (http/https only, private and loopback addresses rejected, DNS-resolved addresses checked), and the Strapi push refuses stale writes. That validation is not a complete boundary, and the broken-link crawler fetches any URL with no internal-address block. The real tool surface, the remote MCP, is shipped fully enabled with no read-only option in the plugin.

- **S L2:** URL validation blocks private/loopback hosts but is not a complete boundary. — [seo/seo-analysis/scripts/push_strapi_seo.py:71-98](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/push_strapi_seo.py#L71-L98) (verified)
  - *To reach the next level:* Validation is not a complete boundary; no host allowlist.
- **C L2:** Seven CMS/preflight scripts define validate_url; the crawler and the MCP path have none. — searched `rg -n 'def validate_url'` in `seo` → 7 hits; searched `rg -n 'is_private|169.254|localhost'` in `seo/broken-link-checker/scripts/checker.py` → 0 hits (verified)
  - *To reach the next level:* The crawler and every MCP tool are outside any shared validation layer.
- **D L0:** The default install wires a single MCP connection with every write capability and no read-only tool set. — [.mcp.json:1-8](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/.mcp.json#L1-L8); [.codex-plugin/plugin.json:27-30](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/.codex-plugin/plugin.json#L27-L30) (verified)
  - *To reach the next level:* No read-only default tool set; write tools are not opt-in.
- **B L1:** A misused tool reaches ad accounts, CMS and CRM broadly; the only quantity bound is the crawler's 50-page default. — [seo/broken-link-checker/scripts/checker.py:159](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/broken-link-checker/scripts/checker.py#L159); [.mcp.json:1-8](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/.mcp.json#L1-L8) (verified)
  - *To reach the next level:* No quantity bounds (spend, recipients, entities) on the write paths.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

The plugin ships Python and shell scripts that the host agent runs directly on the user's machine, as the user, with access to the home directory and gcloud credentials. Nothing in the plugin isolates them, and two skills pre-approve unrestricted Bash so commands they drive run without a host prompt. The upgrade skill pulls the latest main branch and copies it into the plugin cache, so newly fetched code later runs the same way.

- **S L0:** Scripts run as same-user subprocesses on the host via the host's shell; no sandbox primitive exists. — [seo/seo-analysis/scripts/_gcloud.py:26-28](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/_gcloud.py#L26-L28); searched `rg -n -i 'sandbox|seccomp|docker|container'` in `seo/seo-analysis/scripts seo/broken-link-checker/scripts bin .claude-plugin .codex-plugin .mcp.json` → 0 hits (verified)
  - *To reach the next level:* No OS-level isolation for any script the plugin runs.
- **C L0:** No execution path is sandboxed; Bash is pre-approved in two skills. — [seo/setup-cms/SKILL.md:12-13](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/setup-cms/SKILL.md#L12-L13); [notfair-upgrade-skill/SKILL.md:9-10](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/notfair-upgrade-skill/SKILL.md#L9-L10) (verified)
  - *To reach the next level:* Not even the main script path is isolated.
- **D L0:** There is no isolation to enable. — [seo/setup-cms/SKILL.md:12-13](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/setup-cms/SKILL.md#L12-L13) (verified)
  - *To reach the next level:* No isolation on by default.
- **B L0:** Scripts run with the user's full home directory, gcloud ADC tokens and network. — [seo/seo-analysis/scripts/_gcloud.py:35-38](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/_gcloud.py#L35-L38); [notfair-upgrade-skill/SKILL.md:66-81](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/notfair-upgrade-skill/SKILL.md#L66-L81) (verified)
  - *To reach the next level:* Host-equivalent reach: credentials and the full filesystem are available to executed code.
- **Cap:** none

### C5 Untrusted input blast radius — 0.07 (high)

The skills routinely have the agent read content the user did not write: competitor pages, crawled sites, CMS content, search data and MCP results. Nothing in the plugin marks that content as untrusted or separates it from instructions, and the same session holds write access to ad budgets, CRM conversations and CMS content plus open web egress. If injected text hijacks the agent, nothing in the plugin stops it leaking data or making changes; any protection comes from the host's approval prompts, which this plugin does not control.

- **S L1:** Script outputs are plain text/JSON with no provenance or untrusted flag; isolation guidance exists only in prompts. — [seo/broken-link-checker/scripts/checker.py:96-112](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/broken-link-checker/scripts/checker.py#L96-L112); searched `rg -n -i 'untrusted|provenance'` in `seo/seo-analysis/scripts seo/broken-link-checker/scripts bin` → 0 hits (verified)
  - *To reach the next level:* No structured separation of fetched content from metadata and no provenance the host can act on.
- **C L0:** No untrusted source is distinguished. — searched `rg -n -i 'untrusted|provenance'` in `seo/seo-analysis/scripts seo/broken-link-checker/scripts bin` → 0 hits (verified)
  - *To reach the next level:* Fetched pages, CMS content and MCP results all enter context with the same standing.
- **D L0:** No control exists to be on by default. — searched `rg -n -i 'untrusted|provenance'` in `seo/seo-analysis/scripts seo/broken-link-checker/scripts bin` → 0 hits (verified)
  - *To reach the next level:* No default mechanism.
- **B L0:** A hijacked session can exfiltrate (open web fetch, CRM messages) and take irreversible ad-spend or CMS actions; the plugin adds no human step. — [seo/broken-link-checker/scripts/checker.py:96-112](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/broken-link-checker/scripts/checker.py#L96-L112); [paid-ads/shared/operating-contract.md:10](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/paid-ads/shared/operating-contract.md#L10); [.mcp.json:1-8](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/.mcp.json#L1-L8) (verified)
  - *To reach the next level:* Nothing in the plugin forces approval on egress or irreversible actions after untrusted content is read.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.10 (high)

Skills save business context, personas, change logs and content calendars to a local data directory and read them back in later sessions, with no validation or provenance: whatever the model writes becomes trusted context next time. A project-level .notfair.json switches which account and data directory are used. The CMS scripts also auto-load .env and .env.local from the working directory and up to five parent directories, and send the stored API key as a Bearer token to whatever STRAPI_URL those files name, so a cloned project can redirect credentials.

- **S L0:** Model-written context files are re-read as trusted input, and workspace .env files can set the credential-bearing endpoint. — [google-ads/shared/preamble.md:27-40](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/google-ads/shared/preamble.md#L27-L40); [seo/seo-analysis/scripts/push_strapi_seo.py:121-130](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/push_strapi_seo.py#L121-L130); [seo/seo-analysis/scripts/push_strapi_seo.py:140](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/push_strapi_seo.py#L140); [seo/seo-analysis/scripts/push_strapi_seo.py:153](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/push_strapi_seo.py#L153) (verified)
  - *To reach the next level:* No validation, provenance or trust decision before persisted or workspace-supplied context is used.
- **C L0:** Neither the data directory nor workspace config/.env loading is controlled. — [google-ads/shared/preamble.md:27-40](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/google-ads/shared/preamble.md#L27-L40); [seo/seo-analysis/scripts/push_strapi_seo.py:121-130](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/seo-analysis/scripts/push_strapi_seo.py#L121-L130) (verified)
  - *To reach the next level:* No store or auto-loaded file is controlled.
- **D L1:** Data lives in the local user's ~/.notfair, but a project .notfair.json redirects it to a workspace-local .notfair/ directory. — [google-ads/shared/preamble.md:27-40](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/google-ads/shared/preamble.md#L27-L40) (verified)
  - *To reach the next level:* Isolation is chosen by a workspace file rather than enforced.
- **B L1:** Poisoned context persists across the user's sessions and informs later mutations (change-log reviews, business context driving ad changes). — [google-ads/shared/preamble.md:27-40](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/google-ads/shared/preamble.md#L27-L40); [bin/notfair-change-watch:12-16](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/bin/notfair-change-watch#L12-L16) (verified)
  - *To reach the next level:* Poisoned entries persist and can steer tool use rather than being session-scoped or human-reviewed.
- **Cap:** C6-REPOCONFIG — Workspace .env/.env.local files are auto-loaded and can set STRAPI_URL, the endpoint the API key is sent to, without a trust decision.

### C7 Third-party extensions — 0.12 (high)

The plugin wires a remote MCP server whose tools and descriptions are whatever NotFair's server returns at each connection, with nothing pinned. The upgrade skill fetches main, hard-resets the marketplace checkout and copies it into the plugin cache with no signature or hash check, and its inline flow is labelled auto-upgrade. Upgraded code and the MCP tools run with the user's full authority and OAuth grants.

- **S L1:** Upgrades take whatever is on main; the MCP endpoint is an unpinned URL. — [notfair-upgrade-skill/SKILL.md:66-81](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/notfair-upgrade-skill/SKILL.md#L66-L81); [.mcp.json:1-8](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/.mcp.json#L1-L8); searched `rg -n 'sha256|signature|gpg|verify-commit'` in `notfair-upgrade-skill/SKILL.md bin` → 0 hits (verified)
  - *To reach the next level:* No version pin, hash or signature check on upgrades or tool definitions.
- **C L0:** Neither upgrades nor MCP tool definitions are verified. — [notfair-upgrade-skill/SKILL.md:66-81](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/notfair-upgrade-skill/SKILL.md#L66-L81) (verified)
  - *To reach the next level:* No extension type is verified.
- **D L1:** The MCP connection is enabled by installing the plugin; the upgrade flow proceeds without asking once invoked. — [notfair-upgrade-skill/SKILL.md:34-36](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/notfair-upgrade-skill/SKILL.md#L34-L36); [.mcp.json:1-8](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/.mcp.json#L1-L8) (verified)
  - *To reach the next level:* Installation does not show what will run, and upgrades do not ask for consent per version.
- **B L0:** Upgraded scripts run as the same user with the full environment; MCP tools act with the user's OAuth grants across all platforms. — [notfair-upgrade-skill/SKILL.md:66-81](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/notfair-upgrade-skill/SKILL.md#L66-L81); [notfair-upgrade-skill/SKILL.md:9-10](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/notfair-upgrade-skill/SKILL.md#L9-L10) (verified)
  - *To reach the next level:* No per-extension confinement or scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.05 (high)

The setup-cms skill asks the user to paste WordPress application passwords and Strapi full-access tokens into the chat, which puts them in the model's context and the host transcript, then writes them in plaintext to .env.local without restricting permissions. There is no redaction anywhere in the scripts. The plugin ships no telemetry.

- **S L0:** Secrets are routinely collected through chat and stored as plaintext .env.local files. — [seo/setup-cms/SKILL.md:86-95](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/setup-cms/SKILL.md#L86-L95); [seo/setup-cms/SKILL.md:210-219](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/setup-cms/SKILL.md#L210-L219); searched `rg -n -i 'redact|mask'` in `seo/seo-analysis/scripts bin` → 0 hits (verified)
  - *To reach the next level:* No keychain or redaction; secrets pass through model context by design.
- **C L0:** No path (model context, transcripts, files) is protected. — [seo/setup-cms/SKILL.md:86-95](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/setup-cms/SKILL.md#L86-L95) (verified)
  - *To reach the next level:* No path is protected.
- **D L1:** No telemetry is present; nothing redacts by default. — searched `rg -n -i 'telemetry|sentry|posthog'` in `seo/seo-analysis/scripts seo/broken-link-checker/scripts bin` → 0 hits (verified)
  - *To reach the next level:* No redaction exists to be always on.
- **B L0:** Long-lived high-privilege keys (Strapi full-access token, WordPress application password) are reachable by the model and every subprocess. — [seo/setup-cms/SKILL.md:119](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/setup-cms/SKILL.md#L119); [seo/setup-cms/SKILL.md:86-95](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/setup-cms/SKILL.md#L86-L95) (verified)
  - *To reach the next level:* Keys are long-lived and broadly scoped.
- **Cap:** C8-MODELSECRETS — setup-cms routinely collects API keys and application passwords through chat, placing them in model-bound messages.

### C9 Audit & traceability — 0.00 (high)

Plugin code keeps no record of what it did: scripts print progress to stderr and nothing is persisted. The skills ask the model to write change logs and intervention records, but those are model-authored files in a user-writable directory, not an audit trail. Any transcript or server-side history belongs to the host or NotFair's private service.

- **S L0:** No structured record of actions is produced by plugin code. — searched `rg -n 'import logging|getLogger'` in `seo/seo-analysis/scripts seo/broken-link-checker/scripts bin` → 0 hits (verified)
  - *To reach the next level:* No structured per-action record.
- **C L0:** Nothing recorded by plugin code. — searched `rg -n 'import logging|getLogger'` in `seo/seo-analysis/scripts seo/broken-link-checker/scripts bin` → 0 hits (verified)
  - *To reach the next level:* No path is recorded.
- **D L0:** No logging to enable; change logs are model-written by prompt instruction. — [google-ads/manage/references/intervention-memory.md:31-32](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/google-ads/manage/references/intervention-memory.md#L31-L32) (verified)
  - *To reach the next level:* No default-on record outside the model's control.
- **B L0:** Actions proceed with no record. — searched `rg -n 'import logging|getLogger'` in `seo/seo-analysis/scripts seo/broken-link-checker/scripts bin` → 0 hits (verified)
  - *To reach the next level:* Records are not written per action.
- **Cap:** none

### C10 Limits & kill switch — 0.35 (high)

All 20 HTTP calls in the bundled scripts carry timeouts, retries are capped at three, and the crawler stops at 50 pages by default, though the caller can raise that. Nothing in the plugin bounds the consequential path: there is no spend ceiling, rate limit or count limit on MCP mutations. Pausing or stopping is left entirely to the host.

- **S L2:** Every urlopen call has a timeout; crawler page cap and bounded retries exist. — searched `rg -n 'urlopen\('` in `seo bin` → 20 hits (All 20 calls pass timeout=.); [seo/broken-link-checker/scripts/checker.py:159](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/broken-link-checker/scripts/checker.py#L159) (verified)
  - *To reach the next level:* No caps on every operation (output size, rate limits) and no cancellation.
- **C L2:** Timeouts cover all script network calls; the MCP path is unbounded by the plugin. — searched `rg -n 'urlopen\('` in `seo bin` → 20 hits (verified)
  - *To reach the next level:* No limits on the MCP mutation path.
- **D L1:** Defaults exist (50 pages, 10-30s timeouts) but the model, as the caller, can raise the crawl limit via --max-pages. — [seo/broken-link-checker/scripts/checker.py:159](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/seo/broken-link-checker/scripts/checker.py#L159) (verified)
  - *To reach the next level:* The model can raise its own limits; no hard ceiling.
- **B L0:** No plugin-side ceiling on spend or number of mutations. — [paid-ads/shared/operating-contract.md:10](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/paid-ads/shared/operating-contract.md#L10); [.mcp.json:1-8](https://github.com/nowork-studio/notfair-plugin/blob/1a53d4e4da69a6060c24de42c23333fb48e927c4/.mcp.json#L1-L8) (verified)
  - *To reach the next level:* No spend or action ceiling on the consequential path.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Crawled and competitor pages, CMS content, MCP results (seo/broken-link-checker/scripts/checker.py:96-112) · [B] sensitive data/systems: Ad accounts, GA4/GSC, CRM conversations, CMS keys (docs/mcp-connection.md:3-7, seo/setup-cms/SKILL.md:86-95) · [C] state change / egress: MCP mutations to ad spend/CRM/WordPress, Strapi PUT, open web fetch (seo/seo-analysis/scripts/push_strapi_seo.py:153, .mcp.json:5) · Same default session? Yes

## Highest-impact improvements
1. Remove the --yes bypass (or require a human-typed token) in push_strapi_seo.py so the diff confirmation cannot be satisfied by the model. — C2 D L1→L3, +0.100 before caps (Playbook 5)
2. Stop auto-loading .env/.env.local from the working directory and parents; read CMS config only from a user-scope file. — C6 S L0→L2, +0.150 before caps (Playbook 2)
3. Have setup-cms tell users to write keys to the env file themselves (never paste into chat) and chmod 600 the file. — C8 S L0→L2, +0.150 before caps (Playbook 4)
4. Drop pre-approved Bash from setup-cms and upgrade skill frontmatter. — C2 C L0→L1, +0.075 before caps (Playbook 5)
5. Pin upgrades to signed release tags and show the diff/version before installing. — C7 S L1→L3, +0.150 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The NotFair MCP server (https://notfair.co/api/mcp/notfair) is private; its OAuth scopes, tool annotations, approval steps and logging could not be examined and are not credited.
- Host behaviour (Claude Code/Codex/Cursor approval prompts, transcripts) is not credited, per tool-server scoring rules; Claude Code allowed-tools semantics are inferred from host documentation.
- The optional notfair/ local goal-loop app (npx notfair) in the same repo was not scored; its Codex adapter launches codex with --dangerously-bypass-approvals-and-sandbox (notfair/src/server/adapters/codex-local/execute.ts:40) and would warrant a separate audit.
- Skill prose (48 SKILL.md files) was sampled, not read exhaustively; it contains approval guidance that counts as prompt-only.
- No text aimed at AI reviewers or auditors was found in the repository.
