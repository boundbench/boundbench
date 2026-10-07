# Defense-in-Depth Score: GitHub Agentic Workflows

**Repo:** https://github.com/github/gh-aw · **Commit:** `84ebcfe6d35181cc7f0ae2df1220584ed3d92afb` (v0.90.3-96-g84ebcfe6d3) · **Reviewed:** 2026-10-04
**What it is:** GitHub CLI extension that compiles Markdown agentic workflows into GitHub Actions that run Copilot, Claude, Codex, Gemini or Pi agents.
**Category:** Coding
**Scored configuration:** A quick-start workflow (e.g. githubnext/agentics repo-status) compiled by `gh aw compile` with defaults: Copilot engine, strict mode, default network allowlist with AWF firewall, read-only agent job, safe-outputs create-issue with threat detection, on a GitHub-hosted runner.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication yes

## Score: 6.2 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L3 | L2 | L3 | L3 | 0.68 | — | **0.68** | High |
| C2 | Approval gates | L1 | L2 | L0 | L2 | 0.33 | G1 | **0.33** (alt) | High |
| C3 | Tool & action scoping | L2 | L3 | L2 | L3 | 0.62 | — | **0.62** | High |
| C4 | Code-execution isolation | L2 | L3 | L3 | L3 | 0.68 | — | **0.68** | High |
| C5 | Untrusted input blast radius | L2 | L2 | L3 | L1 | 0.50 | — | **0.50** | Medium |
| C6 | Memory, context & configuration integrity | L3 | L2 | L3 | L3 | 0.68 | — | **0.68** | High |
| C7 | Third-party extensions | L3 | L2 | L3 | L2 | 0.62 | — | **0.62** | High |
| C8 | Secrets & sensitive-data protection | L3 | L3 | L3 | L2 | 0.70 | — | **0.70** | High |
| C9 | Audit & traceability | L3 | L3 | L2 | L2 | 0.65 | — | **0.65** | High |
| C10 | Limits & kill switch | L3 | L3 | L3 | L3 | 0.75 | — | **0.75** | High |


gh-aw is built around separating the agent from write access. The agent runs read-only inside a firewalled container with no tokens of its own, and its requested writes are typed, counted and applied by a separate job only after an AI threat check. The main gaps are that no human approves what gets posted by default, and that a manipulated agent can still read repository data and may be able to send it out through allowlisted domains. Keep integrity filtering on, and turn on manual approval or staged mode for sensitive repositories.

## Critical gaps
- None: no criterion is capped and no blast radius is at the worst level.

## Criterion details

### C1 Identity & least privilege — 0.68 (high)

The agent itself never holds write access to GitHub. The compiler rejects any write permission on the agent job, the GitHub tools it gets are forced read-only, and the tokens for the model provider and the GitHub tool server are kept out of the agent's container. Writes the workflow author has configured run later, in a separate job that gets only the write permissions those outputs need, using GitHub's per-job token that expires with the job. What keeps this short of the top is that custom tool servers receive whatever secrets the author hands them, and if a repository has a personal access token saved as GH_AW_GITHUB_TOKEN, it is used silently in place of the scoped per-job token.

- **S L3:** Per-capability credentials: the agent job's GITHUB_TOKEN is read-only, the GitHub MCP server is forced read-only, and write tokens exist only in the separate safe-output job with permissions merged from the configured handlers. — [pkg/workflow/dangerous_permissions_validation.go:72](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/dangerous_permissions_validation.go#L72); [pkg/workflow/mcp_github_config.go:325-326](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/mcp_github_config.go#L325-L326); [pkg/workflow/safe_outputs_permissions.go:156](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/safe_outputs_permissions.go#L156); [.github/workflows/daily-team-status.lock.yml:1808-1809](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/.github/workflows/daily-team-status.lock.yml#L1808-L1809) (verified)
  - *To reach the next level:* Credentials are per-job rather than minted per request with token exchange or downscoping for each write.
- **C L2:** All built-in agent paths use the read-only job identity and the engine/MCP tokens are excluded from the container, but custom MCP servers get whatever secrets the author passes. — [.github/workflows/daily-team-status.lock.yml:952](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/.github/workflows/daily-team-status.lock.yml#L952); [pkg/workflow/awf_command_builder.go:483-489](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/awf_command_builder.go#L483-L489) (verified)
  - *To reach the next level:* Custom MCP servers and other extensions do not go through the same scoped-identity layer.
- **D L3:** Default agent job is read-only and the compiler refuses write scopes unconditionally; writes require explicit safe-outputs configuration by the operator. — [pkg/workflow/dangerous_permissions_validation.go:83](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/dangerous_permissions_validation.go#L83); [.github/workflows/daily-team-status.lock.yml:391-394](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/.github/workflows/daily-team-status.lock.yml#L391-L394); [pkg/workflow/github_token.go:45](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/github_token.go#L45) (verified)
  - *To reach the next level:* A GH_AW_GITHUB_TOKEN PAT, if present, silently replaces the scoped per-job token for MCP reads and safe-output writes.
- **B L3:** A hijacked agent holds a read-only token for one repository that expires with the job; writes are limited to configured safe-output types in that repository. — [.github/workflows/daily-team-status.lock.yml:391-394](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/.github/workflows/daily-team-status.lock.yml#L391-L394); [.github/workflows/daily-team-status.lock.yml:1808-1809](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/.github/workflows/daily-team-status.lock.yml#L1808-L1809); [actions/setup/js/create_issue.cjs:787](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/create_issue.cjs#L787) (verified)
  - *To reach the next level:* Write credentials, when used, are a per-job union of handler permissions rather than per-request credentials revoked after use.
- **Cap:** none

### C2 Approval gates — 0.33 (high)

There is no human approval step on the agent's actions by default. Issues, comments and pull requests the agent asks for are queued, checked against the configured output types and counts, and reviewed by a second AI model looking for threats, but that reviewer is a model, not a person. Code changes still reach the main branch only through a pull request that a human merges. An opt-in setting can require a human to approve each run through a GitHub environment, but that is one approval for the whole run before the agent starts, not a review of each action.

- **default configuration** (default; raw 0.10 → 0.10)
  - **S L0:** No human approves individual safe outputs; the only gate before writes is an LLM threat-detection job. — [pkg/workflow/threat_detection_config.go:119](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/threat_detection_config.go#L119); [docs/src/content/docs/introduction/architecture.mdx:401](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/docs/src/content/docs/introduction/architecture.mdx#L401) (verified)
    - *To reach the next level:* No per-call human approval showing the exact output before it is applied.
  - **C L0:** With no gate, every configured safe output (create issue, comment, PR) is applied once the LLM detector passes. — [actions/setup/js/collect_ndjson_output.cjs:320](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/collect_ndjson_output.cjs#L320) (verified)
    - *To reach the next level:* No human gate covers any consequential action path by default.
  - **D L0:** Human approval is opt-in via on.manual-approval; the default compiles without it. — [pkg/workflow/compiler_activation_outputs.go:267](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/compiler_activation_outputs.go#L267) (verified)
    - *To reach the next level:* Human approval is not on by default.
  - **B L2:** Outputs are bounded by per-type max counts and target-repo allowlists; code changes land as PRs needing a human merge, but issues and comments are published immediately. — [actions/setup/js/collect_ndjson_output.cjs:341](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/collect_ndjson_output.cjs#L341); [actions/setup/js/create_issue.cjs:787](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/create_issue.cjs#L787) (verified)
    - *To reach the next level:* Issues and comments have no preview or dry run by default (staged mode is opt-in).
- **opt-in on.manual-approval GitHub environment** (alt; raw 0.33, cap G1 → 0.33) ← counted
  - **S L1:** An environment protection rule makes a human approve the activation job: one blanket approval per run before the agent executes. — [pkg/workflow/compiler_activation_outputs.go:267](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/compiler_activation_outputs.go#L267) (verified)
    - *To reach the next level:* The approver does not see the exact outputs the agent will produce.
  - **C L2:** Because the whole run waits on the environment, no action path runs without the approval. — [pkg/workflow/compiler_activation_outputs.go:267](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/compiler_activation_outputs.go#L267) (verified)
    - *To reach the next level:* Approval does not cover individual actions after the agent has read untrusted content.
  - **D L0:** Opt-in only. — [pkg/workflow/compiler_activation_outputs.go:267](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/compiler_activation_outputs.go#L267) (verified)
    - *To reach the next level:* Not enabled by default.
  - **B L2:** Same output bounds as the default. — [actions/setup/js/collect_ndjson_output.cjs:341](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/collect_ndjson_output.cjs#L341) (verified)
    - *To reach the next level:* Issues and comments have no preview by default.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C3 Tool & action scoping — 0.62 (high)

The actions that change things outside the sandbox are narrow, typed tools (create an issue, add a comment, open a pull request). They are validated in code: an unknown output type is rejected, each type has a maximum count, target repositories must be on an allowlist, and text is cleaned before posting. Inside the sandbox, though, the default tool set gives the agent an unrestricted shell and file editing, and network access is limited only by a domain allowlist.

- **S L2:** Safe outputs are typed and validated (type allowlist, max counts, repo allowlist, URL-domain sanitization), but the default sandbox tool set includes an unrestricted shell with no argument validation. — [pkg/workflow/copilot_engine_tools.go:95](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/copilot_engine_tools.go#L95); [actions/setup/js/collect_ndjson_output.cjs:320](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/collect_ndjson_output.cjs#L320); [actions/setup/js/create_issue.cjs:787](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/create_issue.cjs#L787) (verified)
  - *To reach the next level:* The general shell tool is not replaced by narrow tools or validated against an allowlist.
- **C L3:** Every built-in safe output passes through the shared NDJSON collector that enforces type, schema and count, and handlers re-validate target repos. — [actions/setup/js/collect_ndjson_output.cjs:341](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/collect_ndjson_output.cjs#L341); [actions/setup/js/create_issue.cjs:652](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/create_issue.cjs#L652) (verified)
  - *To reach the next level:* Custom safe-jobs and custom MCP tools are not covered by the same central policy layer.
- **D L2:** When the sandbox is on (the default), the compiler adds edit and bash with a wildcard; GitHub tools are read-only and writes need explicit safe-outputs configuration. — [pkg/workflow/tools.go:630](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/tools.go#L630); [pkg/workflow/tools.go:624](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/tools.go#L624) (verified)
  - *To reach the next level:* The default tool set still includes write and exec inside the sandbox.
- **B L3:** Misuse is confined to the sandboxed workspace plus a bounded number of typed writes in an allowlisted repository. — [actions/setup/js/collect_ndjson_output.cjs:341](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/collect_ndjson_output.cjs#L341); [actions/setup/js/create_issue.cjs:787](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/create_issue.cjs#L787) (verified)
  - *To reach the next level:* Shell and edit tools are not narrow or reversible operations.
- **Cap:** none

### C4 Code-execution isolation — 0.68 (high)

Everything the agent runs happens inside the Agent Workflow Firewall container, on a private network whose only way out is a proxy that enforces a domain allowlist, on a GitHub runner that is thrown away after the job. The runner's model and GitHub tokens are explicitly kept out of the container, tool servers must run in their own containers, and the run fails rather than falling back to running on the host. Turning the sandbox off needs a dangerously named feature flag and is refused in strict mode, which is on by default. The container's own hardening (user, capabilities, seccomp) is configured in the separate gh-aw-firewall project and could not be verified here. The agent's container can also write to the shared /tmp/gh-aw directory that later host steps read.

- **S L2:** Default runtime is a Docker container launched by rootless AWF with network isolation; hardening details live in the external gh-aw-firewall images and are not visible in this repo. — [pkg/workflow/sandbox_runtime_profile.go:44-49](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/sandbox_runtime_profile.go#L44-L49) (verified)
  - *To reach the next level:* Container hardening (non-root, dropped capabilities, seccomp, read-only root) cannot be verified from this repository; microVM isolation is a preview opt-in.
- **C L3:** The agent CLI and everything it spawns run inside AWF; custom stdio MCP servers must be containerized; host-side git steps disable hooks; AWF failure exits instead of running on the host. — [pkg/workflow/mcp_config_custom.go:130](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/mcp_config_custom.go#L130); [actions/setup/sh/run_awf_with_startup_retries.sh:118](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/sh/run_awf_with_startup_retries.sh#L118); [actions/setup/sh/commit_cache_memory_git.sh:73](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/sh/commit_cache_memory_git.sh#L73) (verified)
  - *To reach the next level:* Not every spawned-process path is shown to be confined with the same profile (for example the MCP gateway holds the Docker socket).
- **D L3:** On by default; disabling requires features.dangerously-disable-sandbox-agent and is rejected under the default strict mode. — [pkg/workflow/sandbox_validation.go:98](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/sandbox_validation.go#L98); [pkg/workflow/strict_mode_permissions_validation.go:125](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/strict_mode_permissions_validation.go#L125); [pkg/workflow/compiler_yaml_policy.go:27-29](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/compiler_yaml_policy.go#L27-L29) (verified)
  - *To reach the next level:* Sandbox policy lives in the workflow file in the repository rather than outside anything a contributor can propose changes to.
- **B L3:** Inside the container: workspace and /tmp/gh-aw writable, no engine or GitHub MCP tokens, egress limited to an allowlist, on an ephemeral runner. — [.github/workflows/daily-team-status.lock.yml:952](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/.github/workflows/daily-team-status.lock.yml#L952); [pkg/constants/constants.go:374](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/constants/constants.go#L374); [.github/workflows/daily-team-status.lock.yml:923](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/.github/workflows/daily-team-status.lock.yml#L923) (verified)
  - *To reach the next level:* Resource limits are not verified, and /tmp/gh-aw is shared read-write with later host steps.
- **Cap:** none

### C5 Untrusted input blast radius — 0.50 (medium)

gh-aw assumes the agent may be manipulated and limits what that achieves: the agent has no write credentials, its requested writes are typed, counted and reviewed by a second model, links to non-allowlisted domains are stripped from posted text, and on public repositories GitHub content from people without write access is filtered out before the model sees it. Only users with write access or above can trigger it by default. A manipulated agent can still read whatever the repository token can read and could try to send it out through one of the allowlisted domains using credentials an attacker plants in the content, and on private repositories the author filter is off by default.

- **S L2:** Structural limits beyond detection: no write credentials in the agent, deferred typed writes, egress allowlist, and deterministic integrity filtering of untrusted authors on public repos; no human approval after untrusted content is read. — [actions/setup/js/determine_automatic_lockdown.cjs:68](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/determine_automatic_lockdown.cjs#L68); [actions/setup/js/sanitize_content_core.cjs:428](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/sanitize_content_core.cjs#L428); [pkg/workflow/role_checks.go:149](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/role_checks.go#L149) (verified)
  - *To reach the next level:* Egress and state changes are not disabled or forced through human approval once untrusted content enters the session.
- **C L2:** Event text is sanitized and GitHub MCP results are integrity-filtered (public repos), but private repos default to min-integrity none and custom MCP/web content is not filtered. — [actions/setup/js/determine_automatic_lockdown.cjs:68](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/determine_automatic_lockdown.cjs#L68) (verified)
  - *To reach the next level:* Not every source is covered: private repositories and custom tool results are unfiltered by default.
- **D L3:** Integrity filtering, sanitization and safe outputs are on by default; weakening requires explicit frontmatter, and strict mode adds checks such as requiring explicit bash when min-integrity is none. — [pkg/workflow/compiler_yaml_policy.go:27-29](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/compiler_yaml_policy.go#L27-L29); [pkg/workflow/strict_mode_network_validation.go:137](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/strict_mode_network_validation.go#L137) (verified)
  - *To reach the next level:* Operator can set min-integrity none or strict: false without a runtime warning to readers of the run.
- **B L1:** A hijacked agent cannot write directly or reach secrets, but can read repository data and may exfiltrate it through allowlisted multi-tenant hosts (for example www.googleapis.com) with attacker-supplied credentials, unattended. — [.github/workflows/daily-team-status.lock.yml:923](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/.github/workflows/daily-team-status.lock.yml#L923); [pkg/workflow/data/ecosystem_domains.json:44](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/data/ecosystem_domains.json#L44) (inferred)
  - *To reach the next level:* Exfiltration through allowlisted hosts is not blocked or gated by a human.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.68 (high)

By default there is no memory between runs; cache and repository memory are opt-in, and when used with threat detection the cache is only saved after detection passes. Instruction files such as AGENTS.md are loaded from the checked-out repository without a prompt, but for pull-request runs gh-aw restores agent configuration folders and root instruction files from the trusted base branch and deletes a PR-supplied .mcp.json. An agent that wants to change these files has to open a pull request, which a human must merge, and changes to protected files trigger a review request. Instruction files nested deeper in a pull-request checkout are not restored.

- **S L3:** Opt-in memory saves are deferred until after threat detection and strict mode rejects repo-scoped cache; security-relevant agent config in PR checkouts is reset to the base branch. — [pkg/workflow/cache_memory.go:69](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/cache_memory.go#L69); [actions/setup/sh/restore_base_github_folders.sh:45](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/sh/restore_base_github_folders.sh#L45); [actions/setup/sh/restore_base_github_folders.sh:74](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/sh/restore_base_github_folders.sh#L74) (verified)
  - *To reach the next level:* No versioned, integrity-protected memory store; instruction files still load silently.
- **C L2:** Root instruction files, agent config folders and .mcp.json are restored for PR checkouts, but only the listed folders and root files. — [actions/setup/sh/restore_base_github_folders.sh:45](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/sh/restore_base_github_folders.sh#L45) (verified)
  - *To reach the next level:* Nested instruction files and other auto-loaded workspace files are not covered.
- **D L3:** No memory by default; caches are keyed per workflow and integrity policy and strict mode refuses repository-wide cache scope. — [pkg/workflow/strict_mode_network_validation.go:192](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/strict_mode_network_validation.go#L192); [pkg/workflow/cache_memory.go:69](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/cache_memory.go#L69) (verified)
  - *To reach the next level:* No retention limit or per-tenant storage separation enforced by gh-aw itself.
- **B L3:** In the default configuration, poisoned context can persist only through a human-merged PR or visible issues, both easy to inspect and remove. — [pkg/workflow/cache_memory.go:69](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/cache_memory.go#L69) (verified)
  - *To reach the next level:* Persistence is not limited to human-reviewed state with rollback for opt-in memory.
- **Cap:** none

### C7 Third-party extensions — 0.62 (high)

The components gh-aw loads by default are GitHub's own (Copilot CLI, the GitHub MCP server, the firewall images) and are pinned: the CLI is checked against a SHA256 checksum and container images are pinned by digest in the compiled workflow. Plugins are checked out at a pinned commit, and custom stdio tool servers must run in their own containers. Custom container images are only pinned when a digest is already known, otherwise they run by tag. Adding an extension requires editing and recompiling the workflow.

- **S L3:** Default components are digest-pinned or checksum-verified; plugins are pinned to commit SHAs; changes require recompiling the lock file. — [.github/workflows/daily-team-status.lock.yml:2](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/.github/workflows/daily-team-status.lock.yml#L2); [actions/setup/sh/install_copilot_cli.sh:24](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/sh/install_copilot_cli.sh#L24); [pkg/workflow/plugin_installation.go:24](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/plugin_installation.go#L24) (verified)
  - *To reach the next level:* No explicit re-approval step when an extension's tool definitions change.
- **C L2:** First-party images and plugins are pinned, but custom MCP container images fall back to an unpinned tag when no digest pin is cached. — [pkg/workflow/docker.go:226](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/docker.go#L226) (verified)
  - *To reach the next level:* Not every extension type is pinned or integrity-checked.
- **D L3:** Nothing third-party is enabled by default; extensions are added only by editing the workflow frontmatter, where the exact container and command appear. — [pkg/workflow/mcp_config_custom.go:130](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/mcp_config_custom.go#L130) (verified)
  - *To reach the next level:* Extensions are declared in repository files rather than a user or admin scope.
- **B L2:** Custom stdio MCP servers run as separate containers launched by the gateway with only the env the author declares. — [pkg/workflow/mcp_config_custom.go:130](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/mcp_config_custom.go#L130) (verified)
  - *To reach the next level:* Per-extension network and filesystem limits are not verified in this repository.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.70 (high)

Credentials are kept out of the agent: the model-provider token is held by a separate API proxy and the GitHub tool token by the MCP gateway, so the model never sees them. Before any logs or outputs are uploaded, files are scanned and known secret values plus common token patterns are redacted, and this runs even when the job fails. Exporting telemetry is off unless the operator sets an endpoint. The Copilot token is still a long-lived personal access token, though narrowly scoped, and an optional GH_AW_GITHUB_TOKEN PAT can be broad.

- **S L3:** Secrets stay in GitHub Actions secrets and out of the agent container; redaction of exact values and token patterns runs before artifacts upload. — [.github/workflows/daily-team-status.lock.yml:952](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/.github/workflows/daily-team-status.lock.yml#L952); [actions/setup/js/redact_secrets.cjs:69](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/redact_secrets.cjs#L69); [actions/setup/js/redact_secrets.cjs:5](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/redact_secrets.cjs#L5) (verified)
  - *To reach the next level:* Engine credential is a long-lived PAT rather than a short-lived token.
- **C L3:** Model-bound context, artifacts, logs and transcripts under /tmp/gh-aw are covered; subprocess env is passed by denylist (--env-all with excludes). — [pkg/workflow/awf_command_builder.go:483-489](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/awf_command_builder.go#L483-L489); [actions/setup/js/redact_secrets.cjs:5](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/actions/setup/js/redact_secrets.cjs#L5) (verified)
  - *To reach the next level:* Environment passed to the container is a denylist, so non-excluded variables still reach it; OTel export redaction not verified.
- **D L3:** Redaction always runs; OTLP export is only configured when the operator supplies an endpoint secret or variable. — [.github/workflows/daily-team-status.lock.yml:79](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/.github/workflows/daily-team-status.lock.yml#L79) (verified)
  - *To reach the next level:* Stored artifacts and transcripts are not encrypted or minimised by default.
- **B L2:** Leaked material would be scoped tokens, but the Copilot PAT and optional GH_AW_GITHUB_TOKEN are long-lived. — [pkg/workflow/github_token.go:45](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/github_token.go#L45) (verified)
  - *To reach the next level:* Not all credentials are short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.65 (high)

Each run leaves a substantial record: GitHub Actions job logs, the agent's own logs, the tool server gateway's logs, firewall and proxy logs of network requests and model usage, and the structured list of outputs the agent asked for, tied together with the run ID and trace attributes. Exporting to an OpenTelemetry collector is supported once an endpoint is configured. Several of these logs sit in /tmp/gh-aw, which the agent's container can write to, so a compromised agent could alter them before they are uploaded. Logging is best-effort and actions do not wait for their record to be written.

- **S L3:** Structured records of tool calls, network requests and safe outputs with run/workflow attribution and OTel trace attributes across jobs. — [.github/workflows/daily-team-status.lock.yml:81](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/.github/workflows/daily-team-status.lock.yml#L81); [.github/workflows/daily-team-status.lock.yml:1170](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/.github/workflows/daily-team-status.lock.yml#L1170) (verified)
  - *To reach the next level:* Records are not tamper-evident: key logs are collected from an agent-writable directory.
- **C L3:** Gateway logs cover MCP tool calls, firewall logs cover network, and safe-output jobs log applied writes. — [.github/workflows/daily-team-status.lock.yml:1170](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/.github/workflows/daily-team-status.lock.yml#L1170) (verified)
  - *To reach the next level:* Configuration changes and credential use are not recorded as audit events.
- **D L2:** On by default and outside the repository workspace, but /tmp/gh-aw (with mcp-logs) is mounted read-write into the agent container. — [pkg/constants/constants.go:374](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/constants/constants.go#L374); [.github/workflows/daily-team-status.lock.yml:1170](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/.github/workflows/daily-team-status.lock.yml#L1170) (verified)
  - *To reach the next level:* Logs are not written by a component the model cannot reach.
- **B L2:** Logs stream to GitHub Actions per step and artifacts upload with if: always(); actions do not depend on records being written. — [.github/workflows/daily-team-status.lock.yml:1170](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/.github/workflows/daily-team-status.lock.yml#L1170) (verified)
  - *To reach the next level:* Records are not durable per action and high-risk actions do not wait for them.
- **Cap:** none

### C10 Limits & kill switch — 0.75 (high)

Runs have firm limits by default: the agent step times out after 20 minutes, a separate API proxy outside the agent caps each run at 1,000 AI credits and 500 model calls, each workflow has a daily cap of 5,000 credits, and every output type has a maximum count. Because the cap is enforced in the proxy, sub-agents and anything else the agent starts count against the same budget, and the model cannot raise it. Cancelling the workflow stops the job through GitHub Actions. These limits can be raised by the operator through repository variables without any hard ceiling.

- **S L3:** Step timeout, per-run AI-credit and invocation caps enforced by the API proxy, daily credit guardrail, and per-type output caps. — [pkg/constants/constants.go:388](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/constants/constants.go#L388); [pkg/constants/constants.go:410](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/constants/constants.go#L410); [pkg/constants/constants.go:420](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/constants/constants.go#L420); [pkg/constants/constants.go:417](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/constants/constants.go#L417) (verified)
  - *To reach the next level:* Halt relies on GitHub Actions cancellation rather than a gh-aw-owned interrupt that is shown to clean up all spawned containers.
- **C L3:** All model traffic from inside the sandbox goes through the API proxy, so sub-agents and spawned processes share the per-run budget. — [pkg/workflow/awf_config.go:178-179](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/workflow/awf_config.go#L178-L179) (verified)
  - *To reach the next level:* No cap on chained workflow dispatches or concurrent delegated runs.
- **D L3:** Defaults are sensible and set outside the container (proxy config, Actions variables); the agent cannot raise them. — [pkg/constants/constants.go:410](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/constants/constants.go#L410); [pkg/constants/constants.go:388](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/constants/constants.go#L388) (verified)
  - *To reach the next level:* No hard ceiling that configuration cannot exceed.
- **B L3:** A runaway is bounded at 20 minutes and 1,000 credits per run; cancelling ends the job. — [pkg/constants/constants.go:410](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/constants/constants.go#L410); [pkg/constants/constants.go:388](https://github.com/github/gh-aw/blob/84ebcfe6d35181cc7f0ae2df1220584ed3d92afb/pkg/constants/constants.go#L388) (verified)
  - *To reach the next level:* Spend ceilings are not enforced provider-side.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Issue/PR/comment content via GitHub MCP and sanitized event text (actions/setup/js/determine_automatic_lockdown.cjs:68) · [B] sensitive data/systems: Repository contents readable with the read-only job token (.github/workflows/daily-team-status.lock.yml:391-394) · [C] state change / egress: Deferred safe-output writes (pkg/workflow/safe_outputs_permissions.go:156) and allowlisted egress (.github/workflows/daily-team-status.lock.yml:923) · Same default session? Yes

## Highest-impact improvements
1. Enable integrity filtering (min-integrity: approved) by default for private repositories too, and filter custom MCP results. — C5 C L2→L3, +0.075 before caps (Playbook 1)
2. Offer a default human review step (staged preview approved before publish) for issues and comments, at least on public repositories. — C2 S L0→L3, +0.225 before caps (Playbook 5)
3. Mount /tmp/gh-aw log directories (mcp-logs, firewall logs) read-only or outside the agent container so a compromised agent cannot rewrite them. — C9 D L2→L3, +0.050 before caps (Playbook 1 step 3)
4. Refuse to resolve custom MCP container images to unpinned tags; require a digest. — C7 C L2→L3, +0.075 before caps (Playbook 3)
5. Pass an explicit env allowlist to the AWF container instead of --env-all with a denylist. — C8 C L3→L4, +0.075 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Agent container hardening (user, capabilities, seccomp), the Squid proxy and the API proxy budget enforcement are implemented in the separate github/gh-aw-firewall project and the gh-aw-mcpg gateway image, which were not reviewed; their behaviour is credited only as configured from this repository.
- The compiled lock file .github/workflows/daily-team-status.lock.yml in this repository was used as a reference for default compiler output; the exact githubnext/agentics repo-status sample was not compiled.
- C5 blast radius (exfiltration through allowlisted multi-tenant hosts) is inferred from the default domain list, not demonstrated.
- Engines other than the default Copilot engine, the Cloud Hypervisor preview runtime and ARC/DinD topologies were not scored.
- No text aimed at steering AI reviewers was found.
