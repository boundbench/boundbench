# Defense-in-Depth Score: PR-Agent

**Repo:** https://github.com/The-PR-Agent/pr-agent · **Commit:** `090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e` · **Reviewed:** 2026-10-03
**What it is:** Open-source AI PR reviewer (GitHub/GitLab/Bitbucket bot, Action, CLI)
**Category:** Coding
**Scored configuration:** GitHub Action as documented in docs/docs/installation/github.md (pull_request and issue_comment triggers, issues/pull-requests/contents/checks write, OPENAI_KEY and GITHUB_TOKEN) with the shipped configuration.toml defaults.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 5.3 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C2 | Approval gates | L0 | L0 | L0 | L2 | 0.10 | — | **0.10** | High |
| C3 | Tool & action scoping | L2 | L3 | L1 | L3 | 0.57 | — | **0.57** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L3 | L2 | L1 | L2 | 0.53 | C5-PUBLICTRIGGER | **0.25** | High |
| C6 | Memory, context & configuration integrity | L2 | L3 | L3 | L3 | 0.68 | — | **0.68** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C9 | Audit & traceability | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |

Controls where a risk surface exists: 3.20 / 8.0 (40%); 2 criteria scored SA (surface absent).

PR-Agent's model has no tools: deterministic code publishes its structured output to the triggering PR, it runs no code, loads no plugins, and reads its configuration only from the default branch. The dominant risk is who can drive it: authorization of comment-triggered commands and of comment-supplied setting overrides is not locked down. Nothing is human-approved before publishing.

## Critical gaps
- None: no criterion is capped and no blast radius is at the worst level.

## Criterion details

### C1 Identity & least privilege — 0.50 (high)

In the GitHub Action, PR-Agent acts with the workflow's GITHUB_TOKEN, a repository-scoped token that lives only for the job. The official workflow grants it write access to contents, pull requests, issues and checks, and one token serves every read and write. Apart from one per-user access check for sibling-repository context files, there is no authorization check in code: whoever can trigger the workflow gets the token's full authority.

- **S L2:** The agent uses one repository-scoped GITHUB_TOKEN for all reads and writes, with permissions fixed by the workflow for the whole job. — [pr_agent/servers/github_action_runner.py:230](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/servers/github_action_runner.py#L230); [docs/docs/installation/github.md:16-36](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/docs/docs/installation/github.md#L16-L36) (official GitHub Action workflow grants issues/pull-requests/contents/checks write) (verified)
  - *To reach the next level:* Read and write share one credential; there is no per-capability token and no in-code authorization gate.
- **C L2:** Every built-in tool reaches GitHub through the same provider client with that token; there are no plugins or sub-agents to widen it, but there is no per-request authorization layer either. — [pr_agent/servers/github_action_runner.py:431-437](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/servers/github_action_runner.py#L431-L437); [pr_agent/git_providers/github_provider.py:1549](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/git_providers/github_provider.py#L1549) (verified)
  - *To reach the next level:* No shared authorization layer checks each action against the requesting principal (only the sibling-repo context read checks the command actor).
- **D L2:** The documented workflow grants contents/pull-requests/issues/checks write even though only opt-in features need contents write; narrowing is an operator workflow edit. — [docs/docs/installation/github.md:16-36](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/docs/docs/installation/github.md#L16-L36) (official GitHub Action workflow grants issues/pull-requests/contents/checks write); [pr_agent/settings/configuration.toml:281](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/settings/configuration.toml#L281) (verified)
  - *To reach the next level:* Default is not read-only; contents: write is granted by the documented workflow although the default tools only need pull-requests/issues write.
- **B L2:** A hijacked or misused token can write to one repository (branches, PRs, issues, checks) for the job's lifetime. — [docs/docs/installation/github.md:16-36](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/docs/docs/installation/github.md#L16-L36) (official GitHub Action workflow grants issues/pull-requests/contents/checks write); [pr_agent/git_providers/github_provider.py:1786-1792](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/git_providers/github_provider.py#L1786-L1792) (verified)
  - *To reach the next level:* Writes are not limited to non-destructive operations; contents: write allows pushing to any unprotected branch.
- **Cap:** none

### C2 Approval gates — 0.10 (high)

PR-Agent has no human approval step. When a PR opens, it automatically rewrites the PR description, posts a review and code suggestions, and sets labels. Any comment command runs straight away, including one that commits a CHANGELOG.md update to the PR branch. Merging and approving are not possible: the approval options are disabled and blocked as comment arguments. Most of what it does can be undone, but comments and notifications go out immediately.

- **S L0:** No approval mechanism exists; outputs are published by deterministic code as soon as the model answers. — searched `rg -n -e 'input\(|confirm|approval'` in `pr_agent` → 23 hits (hits are mosaico user-input reads, the disabled auto-approval option, publish-confirmation helpers, prompt text, and a sibling-repo host-approval docstring; none gates a publish on a human decision); [pr_agent/servers/github_action_runner.py:362-367](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/servers/github_action_runner.py#L362-L367) (verified)
  - *To reach the next level:* No per-call human approval for publishing comments, editing the PR, or committing.
- **C L0:** With no gate, every publish path (comments, description edit, labels, changelog commit) runs ungated. — [pr_agent/git_providers/github_provider.py:525-529](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/git_providers/github_provider.py#L525-L529); [pr_agent/git_providers/github_provider.py:1786-1792](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/git_providers/github_provider.py#L1786-L1792) (verified)
  - *To reach the next level:* No gate exists for any consequential path to traverse.
- **D L0:** There is no approval to turn on. — [pr_agent/servers/github_action_runner.py:362-367](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/servers/github_action_runner.py#L362-L367) (verified)
  - *To reach the next level:* No approval mode exists, on or off.
- **B L2:** Default actions are comments, PR title/body edits, labels and (via comment argument) a CHANGELOG.md commit to the PR branch, all reversible through GitHub history; posted comments and notifications cannot be recalled, and merge/approve are unavailable. — [pr_agent/git_providers/github_provider.py:525-529](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/git_providers/github_provider.py#L525-L529); [pr_agent/git_providers/github_provider.py:1830-1834](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/git_providers/github_provider.py#L1830-L1834); [pr_agent/git_providers/github_provider.py:1786-1792](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/git_providers/github_provider.py#L1786-L1792); [pr_agent/settings/configuration.toml:29](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/settings/configuration.toml#L29) (verified)
  - *To reach the next level:* No preview or dry-run for external actions and no checkpoint beyond git/PR history.
- **Cap:** none

### C3 Tool & action scoping — 0.57 (high)

The model has no tools. Deterministic code decides each action and publishes the model's structured output to the same PR, so the set of actions is narrow and bounded by design. The weak spot is comment arguments: filtering of comment-supplied setting overrides is not a strict boundary. Every command passes through this one filter.

- **S L2:** Actions are narrow, fixed operations (comment, label, edit PR, write CHANGELOG.md at a captured blob SHA), but user-supplied setting overrides are filtered by a substring denylist. — [pr_agent/algo/cli_args.py:218-220](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/algo/cli_args.py#L218-L220); [pr_agent/algo/cli_args.py:174-176](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/algo/cli_args.py#L174-L176); [pr_agent/git_providers/github_provider.py:1786-1792](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/git_providers/github_provider.py#L1786-L1792) (verified)
  - *To reach the next level:* Comment arguments are checked against a denylist, not an allowlist of safe keys with bounds.
- **C L3:** All commands go through _run_command, which validates arguments before applying them; auto-run tools take no user arguments. — [pr_agent/agent/pr_agent.py:340](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/agent/pr_agent.py#L340); [pr_agent/agent/pr_agent.py:351](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/agent/pr_agent.py#L351); [pr_agent/agent/pr_agent.py:49-71](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/agent/pr_agent.py#L49-L71) (verified)
  - *To reach the next level:* Validation is not a central allowlist policy that new settings inherit; new keys are open by default.
- **D L1:** Describe, review and improve run and publish by default; the changelog commit is off by default, but comment-argument handling for write-capable options is not locked down. — [pr_agent/servers/github_action_runner.py:362-367](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/servers/github_action_runner.py#L362-L367); [pr_agent/settings/configuration.toml:281](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/settings/configuration.toml#L281) (verified)
  - *To reach the next level:* Write-capable options are not restricted to operator scope.
- **B L3:** Actions are confined to the triggering PR, with bounded findings/suggestions and a commit limited to CHANGELOG.md on the PR branch. — [pr_agent/tools/pr_update_changelog.py:63](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/tools/pr_update_changelog.py#L63); [pr_agent/git_providers/github_provider.py:1786-1792](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/git_providers/github_provider.py#L1786-L1792); [pr_agent/settings/configuration.toml:266](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/settings/configuration.toml#L266) (verified)
  - *To reach the next level:* Not every operation is reversible (published comments and notifications).
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

PR-Agent never runs model-written or PR-supplied code. The model returns text that is parsed into a fixed review format, and the only subprocesses are fixed-argument shallow git clones. Prompt templates are rendered with Jinja's sandboxed environment, and their source comes from host or default-branch settings that comment arguments cannot change. There is no code-execution surface to isolate.

- **Structural absence:** searched `rg -n -e 'eval\(|exec\(|os\.system|shell=True'` in `pr_agent` → 1 hits (only hit is ast.literal_eval of a webhook command list (bitbucket_server_webhook.py:224), which does not execute code); searched `rg -n 'subprocess\.run\('` in `pr_agent` → 3 hits (fixed-argv `git clone --depth 1` (git_provider.py, bitbucket_server_provider.py) and gerrit `git` helper; argv list, no shell, no model-chosen command; a clone runs no repository hooks)

### C5 Untrusted input blast radius — 0.25 (high)

Untrusted content reaches the model from PR diffs, titles and descriptions, linked issues and comments. Because the model has no tools, a hijacked run can only change the text PR-Agent publishes to that PR; it cannot choose actions or destinations. The bigger gap is authorization of who can drive the bot through comments, which is not locked down.

- **S L3:** The model is tool-less and its output is parsed into a fixed schema that deterministic code publishes to the triggering PR, so injected text cannot choose actions or targets. — [pr_agent/servers/github_action_runner.py:362-367](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/servers/github_action_runner.py#L362-L367); [pr_agent/agent/pr_agent.py:49-71](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/agent/pr_agent.py#L49-L71); [pr_agent/tools/pr_questions.py:255](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/tools/pr_questions.py#L255) (verified)
  - *To reach the next level:* Model-written markdown (links, images) is published unsanitized, and comment commands still select actions.
- **C L2:** All content sources feed the same tool-less model, but access control on comment commands is not locked down. — [pr_agent/servers/github_action_runner.py:431-437](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/servers/github_action_runner.py#L431-L437) (verified)
  - *To reach the next level:* Integrity-protect security-relevant configuration on every path.
- **D L1:** The fixed-plan design is always on, but comment-supplied reconfiguration is not locked down. — [pr_agent/agent/pr_agent.py:351](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/agent/pr_agent.py#L351) (verified)
  - *To reach the next level:* Integrity-protect security-relevant configuration on every path.
- **B L2:** A hijacked run can publish attacker-chosen text, including link or image URLs, and make reversible PR edits or a CHANGELOG commit; the tokens never enter model context, and on public repos only public content is reachable. — [pr_agent/git_providers/github_provider.py:525-529](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/git_providers/github_provider.py#L525-L529); [pr_agent/git_providers/github_provider.py:1786-1792](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/git_providers/github_provider.py#L1786-L1792); [pr_agent/servers/github_action_runner.py:230](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/servers/github_action_runner.py#L230) (verified)
  - *To reach the next level:* Reversible state changes and markdown egress happen unattended, without approval.
- **Cap:** C5-PUBLICTRIGGER — Access control on who can trigger the documented workflow while it holds a write GITHUB_TOKEN is not locked down.

### C6 Memory, context & configuration integrity — 0.68 (high)

PR-Agent loads its settings (.pr_agent.toml) and AGENTS.md context from the repository's default branch, not from the PR, so a pull request cannot rewrite its own reviewer's instructions. Host-only keys such as output sinks, remote config URLs and sibling repositories are stripped from repository settings. The only state it carries between runs is a review-findings marker, read only from comments PR-Agent itself wrote, scoped to that PR and visible in the thread. AGENTS.md is still loaded silently as instruction context.

- **S L2:** Persisted review state is accepted only from the bot's own comments and repo config/context come from the default branch with host-only keys stripped, but AGENTS.md loads silently as instruction context. — [pr_agent/git_providers/git_provider.py:976](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/git_providers/git_provider.py#L976); [pr_agent/git_providers/github_provider.py:1353-1356](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/git_providers/github_provider.py#L1353-L1356); [pr_agent/settings/configuration.toml:36](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/settings/configuration.toml#L36); [pr_agent/config_security.py:22-26](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/config_security.py#L22-L26) (verified)
  - *To reach the next level:* Instruction files load without a trust decision; there is no expiry or approval on what enters context.
- **C L3:** Every persistence and auto-load path is controlled: finding state by authorship, root settings and context files by default-branch reads, per-directory settings off by default and section-restricted. — [pr_agent/tools/pr_reviewer.py:630-635](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/tools/pr_reviewer.py#L630-L635); [pr_agent/settings/configuration.toml:40](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/settings/configuration.toml#L40); [pr_agent/settings/configuration.toml:24](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/settings/configuration.toml#L24) (verified)
  - *To reach the next level:* Retrieved context and summaries are not provenance-tagged end to end.
- **D L3:** State is namespaced per PR in the bot's own comment, and the model has no way to write elsewhere or change scoping. — [pr_agent/tools/pr_reviewer.py:630-635](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/tools/pr_reviewer.py#L630-L635); [pr_agent/algo/review_finding_state.py:15](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/algo/review_finding_state.py#L15) (verified)
  - *To reach the next level:* No retention limit or per-tenant storage separation by default.
- **B L3:** Poisoned state lives in a visible PR comment that only influences later review text for that PR and can be deleted. — [pr_agent/algo/review_finding_state.py:15](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/algo/review_finding_state.py#L15); [pr_agent/git_providers/git_provider.py:976](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/git_providers/git_provider.py#L976) (verified)
  - *To reach the next level:* Persistence is not gated by human review with rollback.
- **Cap:** none

### C7 Third-party extensions — 1.00 (high)

PR-Agent loads no third-party code at runtime. It has no plugin system and no MCP client. The optional 'skills' feature reads only text files from host-configured paths, and the only dynamic imports load the built-in git-provider modules.

- **Structural absence:** searched `rg -n -e 'import_module|entry_points|trust_remote_code|torch\.load|pickle\.load'` in `pr_agent` → 2 hits (both hits are import_module of the built-in git-provider table in git_providers/__init__.py); searched `rg -n -i mcp` in `pr_agent` → 0 hits (no MCP client or server)

### C8 Secrets & sensitive-data protection — 0.38 (high)

Credentials (the GitHub token and model API key) come from environment variables and never enter the prompt; the model has no tool to read them. The code masks credentials in a few places: clone URLs, the /config listing, and merged settings, which it never logs. There is no general log redaction, and git subprocesses inherit the full environment. Telemetry is off by default, and while the default log level is DEBUG, the console sink drops the prompt payloads. The model API key is long-lived.

- **S L1:** Secrets come from env vars, with masking in specific paths (credential-stripped clone URLs, key/secret/token filtering in /config output). — [pr_agent/git_providers/git_provider.py:143-147](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/git_providers/git_provider.py#L143-L147); [pr_agent/tools/pr_config.py:62-70](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/tools/pr_config.py#L62-L70) (verified)
  - *To reach the next level:* No type-level masking or log-wide redaction filter.
- **C L2:** Model-bound messages, /config output and git argv are protected, but subprocess environments are not and logs rely on not logging values. — [pr_agent/git_providers/git_provider.py:264](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/git_providers/git_provider.py#L264); [pr_agent/algo/utils.py:809](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/algo/utils.py#L809) (verified)
  - *To reach the next level:* Subprocess environments get os.environ in full and error/log paths have no redaction filter.
- **D L2:** OTEL telemetry is opt-in; the default DEBUG level is console-only and the console sink omits the prompt artifacts. — [pr_agent/settings/configuration.toml:519](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/settings/configuration.toml#L519); [pr_agent/settings/configuration.toml:20](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/settings/configuration.toml#L20); [pr_agent/log/__init__.py:72-74](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/log/__init__.py#L72-L74); [pr_agent/algo/ai_handlers/litellm_ai_handler.py:2927](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/algo/ai_handlers/litellm_ai_handler.py#L2927) (verified)
  - *To reach the next level:* Redaction is not always-on and content-bearing logs are one setting away.
- **B L1:** A leaked OPENAI_KEY is long-lived and billable; the GITHUB_TOKEN is repo-scoped and job-lived. — [pr_agent/servers/github_action_runner.py:230](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/servers/github_action_runner.py#L230) (verified)
  - *To reach the next level:* Model provider key is not short-lived or task-scoped.
- **Cap:** none

### C9 Audit & traceability — 0.38 (high)

Each command is logged to stdout with its command name and PR URL, along with every setting a comment changed; GitHub keeps the Action log outside anything the agent can write. The console logs are unstructured, and the code does not record who requested a command. Everything PR-Agent publishes is also visible in the PR's own GitHub history. If logging fails, the action carries on regardless.

- **S L1:** Unstructured console logs record command start and applied settings; the requesting user is not recorded. — [pr_agent/agent/pr_agent.py:408-409](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/agent/pr_agent.py#L408-L409); [pr_agent/algo/utils.py:809](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/algo/utils.py#L809); [pr_agent/log/__init__.py:74](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/log/__init__.py#L74) (verified)
  - *To reach the next level:* No structured per-action record with actor attribution in the default Action mode.
- **C L2:** All built-in commands pass through the same logged handler; no extensions exist. — [pr_agent/agent/pr_agent.py:408-409](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/agent/pr_agent.py#L408-L409); [pr_agent/agent/pr_agent.py:49-71](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/agent/pr_agent.py#L49-L71) (verified)
  - *To reach the next level:* Approvals/denials and requester identity are not recorded.
- **D L2:** Logging is on by default to the runner's stdout, which the model cannot reach. — [pr_agent/log/__init__.py:74](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/log/__init__.py#L74); [pr_agent/settings/configuration.toml:20](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/settings/configuration.toml#L20) (verified)
  - *To reach the next level:* Not a dedicated audit component with logged disablement; level is configurable.
- **B L1:** Logging is best-effort; failures neither surface nor block actions. — [pr_agent/log/__init__.py:74](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/log/__init__.py#L74) (verified)
  - *To reach the next level:* Records are not flushed durably per action with errors surfaced.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

Each model call has a 120-second timeout, input is capped at 32k tokens, and chunked tools stop after a few calls. However, these values are not fully protected from comment arguments, there is no spend ceiling, and nothing limits how often commands run. Every new comment starts a fresh run with a fresh budget. Cancelling the GitHub workflow stops the run.

- **S L2:** Per-call timeout, input-token cap and per-tool call counts are enforced in code. — [pr_agent/algo/ai_handlers/litellm_ai_handler.py:2715](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/algo/ai_handlers/litellm_ai_handler.py#L2715); [pr_agent/settings/configuration.toml:47](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/settings/configuration.toml#L47); [pr_agent/settings/configuration.toml:266](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/settings/configuration.toml#L266); searched `rg -n -i 'max_cost|cost_limit|spend_limit|budget_usd|max_budget'` in `pr_agent` → 0 hits (no spend ceiling) (verified)
  - *To reach the next level:* No per-run wall-clock or cost cap and no rate limit on side-effecting commands.
- **C L2:** Limits apply to every model call and the git clone has a timeout; there are no sub-agents. — [pr_agent/algo/ai_handlers/litellm_ai_handler.py:2715](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/algo/ai_handlers/litellm_ai_handler.py#L2715); [pr_agent/settings/configuration.toml:30](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/pr_agent/settings/configuration.toml#L30) (verified)
  - *To reach the next level:* Separate comment-triggered runs do not share a budget.
- **D L1:** Defaults are sensible, but they are not fully protected from comment arguments. (verified)
  - *To reach the next level:* Limits have no hard ceiling.
- **B L1:** Repeated comments trigger unlimited runs against the operator's model key with no spend ceiling. — searched `rg -n -i 'max_cost|cost_limit|spend_limit|budget_usd|max_budget'` in `pr_agent` → 0 hits (no spend ceiling); [docs/docs/installation/github.md:16-19](https://github.com/The-PR-Agent/pr-agent/blob/090a7e4ad7d3adaa8e87b4a0f874a3a4c438139e/docs/docs/installation/github.md#L16-L19) (verified)
  - *To reach the next level:* No tight per-run or per-PR cost ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: PR diff/title/body and issue_comment commands (pr_agent/servers/github_action_runner.py:376-442) · [B] sensitive data/systems: Repository content of private repos and the operator's LLM key/GITHUB_TOKEN held in-process but never in model context (github_action_runner.py:224-230) · [C] state change / egress: PR comments, PR title/body edits, labels and CHANGELOG.md commits (github_provider.py:525-529, 1786-1792) · Same default session? Yes

## Highest-impact improvements
1. Lock down authorization of comment-triggered commands. — C5 C L2→L4, +0.150 before caps (Playbook 1)
2. Replace the comment-argument denylist with an allowlist of safe presentation keys, keeping other keys operator-only. — C3 S L2→L3, +0.075 before caps (Playbook 3)
3. Add hard per-run and per-PR ceilings on model calls and cost, and rate-limit comment commands per PR. — C10 S L2→L3, +0.075 before caps (Playbook 3 step 3)
4. Drop contents: write from the documented workflow unless the changelog push is enabled. — C1 D L2→L3, +0.050 before caps (Playbook 4)
5. Scrub git subprocess environments to the SSL/config variables they need. — C8 C L2→L3, +0.075 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the GitHub Action mode (README's recommended path, using the full workflow from the installation guide). The GitHub App, GitLab/Bitbucket/Azure/Gitea webhook servers, the CLI, and the mosaico server were reviewed only where they share code paths. One App-mode configuration concern was not verified end to end.
- Prompt-injection behaviour of specific models was not assessed; C5 assumes the hijack succeeds.
- No reviewer-injection attempt was found in the repository (AGENTS.md is ordinary contributor guidance).
