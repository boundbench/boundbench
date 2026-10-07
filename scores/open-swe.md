# Defense-in-Depth Score: Open SWE

**Repo:** https://github.com/langchain-ai/open-swe · **Commit:** `8e0e13e33a94da2266e180b0ffeb7c6264832457` · **Reviewed:** 2026-10-03
**What it is:** Open-source async coding agent / software factory built on Deep Agents
**Category:** Coding
**Scored configuration:** Team deployment per docs/INSTALLATION.md with default env (SANDBOX_TYPE=langsmith, GitHub App with documented permissions, Slack app), coding threads started by members from Slack/dashboard/GitHub.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents yes · external communication yes

## Score: 4.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | High |
| C2 | Approval gates | L3 | L0 | L3 | L1 | 0.42 | C2-POWERBYPASS | **0.25** | High |
| C3 | Tool & action scoping | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C4 | Code-execution isolation | L4 | L3 | L2 | L1 | 0.68 | — | **0.68** | High |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C7 | Third-party extensions | L1 | L1 | L2 | L3 | 0.40 | — | **0.40** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C9 | Audit & traceability | L3 | L2 | L3 | L1 | 0.57 | — | **0.57** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


Open SWE runs its shell in a remote cloud sandbox and keeps GitHub tokens out of the sandbox, and only registered users can start runs. But there is no approval step for ordinary actions: a coding thread holds write access to every repository in the GitHub App installation, sandboxes have open network egress, and only workflow-file pushes ask a human, through a guard that does not cover every path. A prompt injection from a web page, repository file or integration result can therefore push code across repositories and send data out without anyone confirming.

## Critical gaps
- The shell tool runs without human approval; the only gate covers workflow-file pushes, and it does not cover every path. (ASI02, ASI09; C2) — [agent/middleware/workflow_push_guard.py:574-584](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/middleware/workflow_push_guard.py#L574-L584)
- A hijacked session can both exfiltrate (open sandbox egress, http_request, Slack) and push to every installation repository unattended (C5 worst case). (ASI01, LLM01; C5) — [agent/server.py:1679-1683](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/server.py#L1679-L1683); [docs/reference/workspaces.md:175-176](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/docs/reference/workspaces.md#L175-L176); [agent/github/comments.py:149-159](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/github/comments.py#L149-L159)

## Criterion details

### C1 Identity & least privilege — 0.33 (high)

Open SWE acts through its own GitHub App rather than a person's account, mints short-lived installation tokens per sandbox, and keeps the GitHub token out of the sandbox by having the LangSmith proxy inject it on the wire. However, ordinary coding threads started from Slack, the dashboard or Linear get a token for every repository the installation can reach with all of the app's write permissions (contents, pull requests, issues, workflows), and the docs state that a member can reach repositories they cannot access themselves. Only threads triggered by public-repository events and the reviewer are narrowed to one repository. Open SWE's own sensitive tools (admin, settings, SQL) do pass a per-actor policy check, and PRs are opened with the requesting user's OAuth token.

- **S L1:** The default coding sandbox gets an installation-wide GitHub App token carrying all app permissions, narrowed to one repository only for public-repo events and reviewer flows. — [agent/github/sandbox_access.py:145-163](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/github/sandbox_access.py#L145-L163); [agent/sandboxes/lifecycle.py:194-196](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/sandboxes/lifecycle.py#L194-L196); [agent/github/token_scope.py:15-23](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/github/token_scope.py#L15-L23); [docs/INSTALLATION.md:84-91](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/docs/INSTALLATION.md#L84-L91) (verified)
  - *To reach the next level:* Default coding threads need a token narrowed to the thread's repositories and to the permissions the task needs.
- **C L2:** Built-in tools use the app identity and Open SWE's own sensitive tools are re-checked per actor by access.py, but GitHub operations through the sandbox shell are never authorized against the requesting principal. — [agent/tools/access.py:224-253](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/tools/access.py#L224-L253); [agent/tools/access.py:256-258](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/tools/access.py#L256-L258); [docs/reference/workspaces.md:175-176](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/docs/reference/workspaces.md#L175-L176); [agent/tools/open_pull_request.py:66-76](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/tools/open_pull_request.py#L66-L76) (verified)
  - *To reach the next level:* GitHub actions from the sandbox are not checked against what the requesting person may access; there is no single authorization layer over every tool path.
- **D L1:** A narrower default exists only for public-repository event threads; every member-started coding thread gets the installation-wide token by default. — [agent/sandboxes/lifecycle.py:194-196](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/sandboxes/lifecycle.py#L194-L196); [agent/github/sandbox_access.py:145-163](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/github/sandbox_access.py#L145-L163); [docs/reference/workspaces.md:175-176](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/docs/reference/workspaces.md#L175-L176) (verified)
  - *To reach the next level:* Default coding threads should start repository-scoped and read-mostly, with write scope an explicit elevation.
- **B L1:** A hijacked coding thread can write to every repository in the installation (including workflow files) and post to Slack and configured integrations. — [docs/INSTALLATION.md:84-91](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/docs/INSTALLATION.md#L84-L91); [docs/reference/workspaces.md:175-176](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/docs/reference/workspaces.md#L175-L176); [agent/server.py:1679-1683](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/server.py#L1679-L1683) (verified)
  - *To reach the next level:* Blast radius should be limited to one project/repository with non-destructive writes.
- **Cap:** none

### C2 Approval gates — 0.25 (high)

There is no general human approval step: shell commands, pushes, Slack posts, outbound HTTP requests and integration tools all run without asking anyone. The one deterministic gate is for git pushes that change GitHub Actions workflow files; it shows the exact diff, binds the approval to a fingerprint and pushes exactly the approved commit. But the guard does not cover every push and write path, and its approver authorization is not locked down. Merges through the human-review and expedited-review flows do require human votes.

- **S L3:** The workflow-push gate shows the exact workflow diff and file list, records approve/reject, and replaces the command with a fixed refspec for the approved head SHA. — [agent/middleware/workflow_push_guard.py:594-600](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/middleware/workflow_push_guard.py#L594-L600); [agent/middleware/workflow_push_guard.py:500-505](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/middleware/workflow_push_guard.py#L500-L505) (verified)
  - *To reach the next level:* No argument-level allow/deny policy beyond workflow-file detection.
- **C L0:** The shell tool is exempt except for workflow-file pushes, and the push guard does not cover every path. — [agent/middleware/workflow_push_guard.py:574-584](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/middleware/workflow_push_guard.py#L574-L584); searched `rg -n "HumanInTheLoopMiddleware|interrupt_on"` in `agent` → 0 hits (No generic human-in-the-loop interrupt anywhere in the agent package.) (verified)
  - *To reach the next level:* Every mutating shell, git and API path would need to traverse the gate, failing closed.
- **D L3:** The guard is always in the main and sub-agent middleware stacks with no flag to disable it, and approvals are stored per exact fingerprint. — [agent/server.py:2042](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/server.py#L2042); [agent/threads/workflow_approval.py:161-177](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/threads/workflow_approval.py#L161-L177) (verified)
  - *To reach the next level:* Approver authorization is not locked down.
- **B L1:** A wrongly allowed action can push to any installation repository, post Slack messages, and send arbitrary HTTP requests; only git history offers partial undo. — [agent/server.py:1679-1683](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/server.py#L1679-L1683); [docs/INSTALLATION.md:84-91](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/docs/INSTALLATION.md#L84-L91) (verified)
  - *To reach the next level:* No checkpoints or rate limits on consequential actions, and external posts have no preview.
- **Cap:** C2-POWERBYPASS — The shell tool, the most powerful action path, runs without approval in the default configuration except for workflow-file pushes.

### C3 Tool & action scoping — 0.25 (high)

The agent's main tool is an unrestricted shell in the sandbox, with `gh` pre-authenticated through the proxy, so most actions are raw command strings. A few server-side tools are carefully bounded: `http_request`/`fetch_url` resolve hosts and refuse non-public addresses on every redirect, MCP connections are pinned to their HTTPS origin, and the admin SQL tool runs in a read-only transaction with row and time limits. Tool availability is filtered per thread by an access policy, but write, exec and network tools are all on by default.

- **S L1:** The dominant tool is a raw shell string; validation exists only in a few narrow tools (SSRF-safe HTTP, read-only SQL) and pattern-based guards on push/PR-creation commands. — [agent/server.py:1679-1683](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/server.py#L1679-L1683); [agent/utils/url_safety.py:15-45](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/utils/url_safety.py#L15-L45); [agent/tools/read_only_sql.py:43-58](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/tools/read_only_sql.py#L43-L58) (verified)
  - *To reach the next level:* Replace shell-driven GitHub operations with narrow, validated tools and argument allowlists.
- **C L1:** Only a few tools (HTTP fetch, MCP transport, SQL) validate their inputs; the shell and integration tools do not. — [agent/tools/http_request.py:17-46](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/tools/http_request.py#L17-L46); [agent/mcp/transport.py:11-29](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/mcp/transport.py#L11-L29); [agent/middleware/workflow_push_guard.py:574-584](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/middleware/workflow_push_guard.py#L574-L584) (verified)
  - *To reach the next level:* Most built-in tools would need validation, with a shared layer for extension tools.
- **D L1:** Shell, file write, outbound HTTP, Slack and PR tools are bound by default; the access policy removes only admin/private-surface tools. — [agent/server.py:1679-1683](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/server.py#L1679-L1683); [agent/server.py:1739](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/server.py#L1739); [agent/tools/access.py:256-258](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/tools/access.py#L256-L258) (verified)
  - *To reach the next level:* Tool groups should be selectable per task with write/exec off unless needed.
- **B L1:** A misused shell or HTTP tool reaches any public host and every installation repository. — [docs/reference/workspaces.md:175-176](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/docs/reference/workspaces.md#L175-L176); searched `rg -n -i "egress|allow_hosts|allowed_hosts|network_policy"` in `agent/sandboxes` → 0 hits (No network egress restriction is configured for sandboxes; the proxy config only injects headers.) (verified)
  - *To reach the next level:* Scope tools to the thread's repository and bound quantities.
- **Cap:** none

### C4 Code-execution isolation — 0.68 (high)

By default every shell command runs in a remote LangSmith cloud sandbox, not on the server that holds Open SWE's keys, and an unreachable sandbox is not silently replaced with host execution. Operators can switch to a documented `local` provider with no isolation, and the desktop app and CLI bridge deliberately run commands on the user's machine. Inside the default sandbox, network egress is unrestricted, the GitHub proxy will attach an installation-wide token to any github.com request, a capability header lets code in the sandbox invoke the thread's own agent tools, and sandboxes persist per thread for up to 30 days.

- **S L4:** Commands execute on a remote sandbox service through the LangSmith sandbox API, a separate machine from the agent server. — [agent/sandboxes/providers/langsmith.py:531-606](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/sandboxes/providers/langsmith.py#L531-L606); [agent/config.py:388-392](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/config.py#L388-L392) (verified)
- **C L3:** Every model-reachable command path (execute, background_execute, workspace scripts) goes to the thread's sandbox, and an unreachable sandbox fails rather than falling back; local, desktop and CLI-bridge modes are documented escape hatches, and the admin-only read-only SQL tool runs server-side. — [agent/server.py:1377-1386](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/server.py#L1377-L1386); [agent/sandboxes/lifecycle.py:466-468](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/sandboxes/lifecycle.py#L466-L468); [README.md:35](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/README.md#L35); [agent/tools/read_only_sql.py:43-58](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/tools/read_only_sql.py#L43-L58) (verified)
  - *To reach the next level:* Remove or explicitly gate the non-sandboxed paths (local provider, bridge, server-side SQL) so no model-reachable path runs outside the sandbox.
- **D L2:** The remote sandbox is the default, but setting SANDBOX_TYPE=local switches to unisolated host execution with only a docstring warning. — [agent/config.py:388-392](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/config.py#L388-L392); [agent/sandboxes/providers/local.py:38-65](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/sandboxes/providers/local.py#L38-L65) (verified)
  - *To reach the next level:* Disabling isolation should require a loudly named flag with a runtime warning.
- **B L1:** Inside the sandbox, code has unrestricted network egress, can use the installation-wide GitHub token through the proxy and can call the thread's agent tools through an injected capability; sandboxes persist across runs. — searched `rg -n -i "egress|allow_hosts|allowed_hosts|network_policy"` in `agent/sandboxes` → 0 hits (No network egress restriction is configured for sandboxes; the proxy config only injects headers.); [agent/sandboxes/providers/langsmith.py:199-212](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/sandboxes/providers/langsmith.py#L199-L212); [agent/sandboxes/tool_access.py:102-113](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/sandboxes/tool_access.py#L102-L113); [agent/sandboxes/tool_routes.py:97-108](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/sandboxes/tool_routes.py#L97-L108) (verified)
  - *To reach the next level:* Egress should be allowlisted and credentials/tool capabilities withheld from sandboxed code.
- **Cap:** none

### C5 Untrusted input blast radius — 0.25 (high)

Only known Open SWE users can trigger runs from GitHub, and comments by unregistered GitHub users are wrapped in a 'dangerous external untrusted' tag with a prompt telling the model to ignore their instructions. Nothing in code acts on that tag, and web pages, repository files, Slack messages and MCP results enter the context with no marking at all. A hijacked agent can therefore push code to any installation repository and exfiltrate data through outbound HTTP or Slack without any human step.

- **S L1:** Untrusted GitHub comments are fenced with delimiter tags and a prompt instruction; no code limits what happens after untrusted content is read. — [agent/github/comments.py:149-159](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/github/comments.py#L149-L159); [agent/resources/prompts/system/external-untrusted-comments.md:5](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/resources/prompts/system/external-untrusted-comments.md#L5) (verified)
  - *To reach the next level:* Once untrusted content enters a session, egress and state-changing tools would need to be disabled or forced through approval.
- **C L1:** Only GitHub comment authors are distinguished; tool results, web fetches, repository files and MCP outputs are not. — searched `rg -n -i "untrusted"` in `agent/tools agent/mcp agent/slack/tools` → 0 hits (Tool, MCP and Slack-read results carry no untrusted/provenance marking.); [agent/github/webhook.py:930-932](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/github/webhook.py#L930-L932) (verified)
  - *To reach the next level:* Every untrusted source (fetch, files, MCP, Slack) would need to fall under the same handling.
- **D L2:** The fencing is applied unconditionally for unregistered authors and cannot be switched off by content. — [agent/github/comments.py:149-159](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/github/comments.py#L149-L159) (verified)
  - *To reach the next level:* No warned, explicit control over it because there is no enforcement to configure.
- **B L0:** A hijacked session can push to every installation repository and send data to any public URL or Slack channel unattended. — [agent/server.py:1679-1683](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/server.py#L1679-L1683); searched `rg -n -i "egress|allow_hosts|allowed_hosts|network_policy"` in `agent/sandboxes` → 0 hits (No network egress restriction is configured for sandboxes; the proxy config only injects headers.); [docs/reference/workspaces.md:175-176](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/docs/reference/workspaces.md#L175-L176) (verified)
  - *To reach the next level:* At least one of exfiltration or irreversible action would need a human step.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.38 (high)

The model can rewrite the user's personal standing instructions and personal skills with a tool call, without a human confirming the text; those are then loaded as trusted guidance into every later session of that user. Writes are limited to the owner on private threads and are audit-logged, and data is namespaced per user. Repository `AGENTS.md` files (and nested ones after file reads) are loaded silently and the system prompt tells the model they override its defaults. Repository files cannot add tools, MCP servers or approval rules.

- **S L1:** save_user_instructions and save_user_skill persist model-written text with only an audit record, and AGENTS.md is loaded as mandatory rules. — [agent/tools/save_user_instructions.py:17-41](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/tools/save_user_instructions.py#L17-L41); [agent/resources/prompts/system/repository-setup.md.jinja:11](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/resources/prompts/system/repository-setup.md.jinja#L11); [agent/middleware/subdir_agents.py:1](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/middleware/subdir_agents.py#L1) (verified)
  - *To reach the next level:* Persistent memory writes should require human approval or validation, and instruction files should carry provenance.
- **C L2:** User instructions/skills are restricted to the owner on private threads by the access policy, but repository AGENTS.md files and nested ones are not controlled. — [agent/tools/user_skills.py:19-27](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/tools/user_skills.py#L19-L27); [agent/middleware/subdir_agents.py:1](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/middleware/subdir_agents.py#L1) (verified)
  - *To reach the next level:* Auto-loaded repository instruction files would need the same control.
- **D L2:** Instructions and skills are keyed by the resolved GitHub login, not a model argument, so writes stay in the user's own namespace. — [agent/tools/save_user_instructions.py:17-41](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/tools/save_user_instructions.py#L17-L41); [agent/tools/user_skills.py:19-27](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/tools/user_skills.py#L19-L27) (verified)
  - *To reach the next level:* Retention limits and per-tenant storage separation are absent (and D is limited by the weak write control).
- **B L1:** Poisoned user instructions persist across all of that user's later sessions and can steer tool use. — [agent/tools/save_user_instructions.py:17-41](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/tools/save_user_instructions.py#L17-L41); [langgraph.json:19-24](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/langgraph.json#L19-L24) (verified)
  - *To reach the next level:* Persisted guidance should only influence gated actions or require review before it takes effect.
- **Cap:** none

### C7 Third-party extensions — 0.40 (high)

Open SWE loads no third-party code in its own process: integrations are remote MCP servers over HTTPS that an admin or user adds explicitly, and requests are pinned to the configured origin and to public IP addresses. Admins choose which of a server's tools are allowed, but tool definitions are re-fetched on a short cache window with no re-approval when they change, and a remote server cannot be version-pinned. A malicious MCP server only receives its own credentials and the arguments sent to it.

- **S L1:** MCP servers are user/admin-chosen remote endpoints with a tool-name allowlist; definitions are refreshed without change detection. — [agent/mcp/runtime.py:242-256](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/mcp/runtime.py#L242-L256); [agent/mcp/runtime.py:216-240](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/mcp/runtime.py#L216-L240) (verified)
  - *To reach the next level:* Pin or hash tool definitions and re-approve on change.
- **C L1:** The name allowlist applies to configured MCP connections; skills and other loaded content have no verification. — [agent/mcp/runtime.py:242-256](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/mcp/runtime.py#L242-L256) (verified)
  - *To reach the next level:* Verification would need to cover all extension types.
- **D L2:** No MCP servers are enabled by default; admins and users add them explicitly and select tools. — [agent/mcp/transport.py:1](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/mcp/transport.py#L1) (verified)
  - *To reach the next level:* Show and approve exact tool definitions; D is limited by the weak verification mechanism.
- **B L3:** An MCP server runs on its own remote host and receives only its own encrypted-at-rest credentials, never the agent's other secrets. — [agent/mcp/transport.py:11-29](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/mcp/transport.py#L11-L29) (verified)
  - *To reach the next level:* No network/file limits on what the server itself declares.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.50 (high)

Stored GitHub, Slack and MCP credentials are encrypted at rest with a rotatable key, and the GitHub token reaches the sandbox only as an opaque proxy-injected header, so the model never sees it. Usage telemetry to Segment is off unless a key is set. There is no redaction of secrets in tool outputs before they reach the model, transcripts or LangSmith traces, and the installation token is broad even though it is short-lived.

- **S L2:** Credentials are Fernet-encrypted at rest and GitHub tokens are injected by the proxy rather than placed in the sandbox environment, but no redaction runs on model-bound content or traces. — [agent/encryption.py:1-45](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/encryption.py#L1-L45); [agent/sandboxes/providers/langsmith.py:199-212](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/sandboxes/providers/langsmith.py#L199-L212) (verified)
  - *To reach the next level:* Redaction before logs and model-bound messages on all major paths.
- **C L2:** Stored credentials and the sandbox environment are protected; model-bound tool output, transcripts and traces are not redacted. — [agent/sandboxes/providers/langsmith.py:199-212](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/sandboxes/providers/langsmith.py#L199-L212); [agent/middleware/transcript.py:1-12](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/middleware/transcript.py#L1-L12) (verified)
  - *To reach the next level:* Model-bound messages and telemetry/traces need protection too.
- **D L2:** Segment usage telemetry is opt-in; tracing follows the deployment's LangSmith settings. — [agent/config.py:344](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/config.py#L344) (verified)
  - *To reach the next level:* Redaction should be always on.
- **B L2:** A leaked installation token expires within the hour but covers every installation repository with write permissions. — [agent/github/sandbox_access.py:145-163](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/github/sandbox_access.py#L145-L163); [docs/INSTALLATION.md:84-91](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/docs/INSTALLATION.md#L84-L91) (verified)
  - *To reach the next level:* Tokens should be both short-lived and narrowly scoped.
- **Cap:** none

### C9 Audit & traceability — 0.57 (high)

Every model call and tool call is written to an append-only transcript event log in Postgres by middleware that also covers sub-agents, alongside LangGraph thread state and LangSmith traces, and workflow-push approvals record who decided. A separate metadata-only audit log covers dashboard writes and a few settings tools. Both are explicitly best-effort: transcript failures are swallowed, audit writes can be lost, and GitHub API calls made from the sandbox through the proxy are only visible as the shell command that issued them.

- **S L3:** Structured transcript events per tool call with run/thread/subagent context, plus audit records with actor attribution and recorded approvers. — [agent/middleware/transcript.py:1-12](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/middleware/transcript.py#L1-L12); [agent/middleware/transcript.py:886](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/middleware/transcript.py#L886); [agent/threads/workflow_approval.py:161-177](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/threads/workflow_approval.py#L161-L177); [docs/reference/audit-logs.md:15](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/docs/reference/audit-logs.md#L15) (verified)
  - *To reach the next level:* No tamper-evident storage (hash chain, signing) for the record.
- **C L2:** Tool calls, sub-agents and approvals are recorded, but actions performed by sandbox code through the GitHub proxy or the sandbox tool capability are not individually attributed. — [agent/middleware/transcript.py:886](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/middleware/transcript.py#L886); [agent/sandboxes/tool_routes.py:97-108](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/sandboxes/tool_routes.py#L97-L108) (verified)
  - *To reach the next level:* All tool paths including sandbox-originated calls should be recorded.
- **D L3:** Recording is on by default and written by server-side middleware the model and sandbox cannot reach. — [agent/middleware/transcript.py:1-12](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/middleware/transcript.py#L1-L12) (verified)
  - *To reach the next level:* Disabling or altering it is not itself logged.
- **B L1:** Transcript and audit writes are best-effort, failures are logged and swallowed, and events can be lost on crash. — [agent/middleware/transcript.py:1-12](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/middleware/transcript.py#L1-L12); [docs/reference/audit-logs.md:32](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/docs/reference/audit-logs.md#L32) (verified)
  - *To reach the next level:* Records should be flushed durably per action with surfaced errors.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

Runs are bounded only loosely: 5,000 model calls and a 9,999-step recursion limit per run, a 15-minute cap per model call, and per-command timeouts, with background commands allowed for up to 24 hours (four at a time). There is no cost or spend ceiling. Stopping a thread interrupts its LangGraph runs and clears queued work, but background commands keep running inside the persistent sandbox until it idles out.

- **S L2:** Model-call cap plus per-call and per-command timeouts are enforced in code; no token/cost cap. — [agent/runtime/constants.py:4-6](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/runtime/constants.py#L4-L6); [agent/server.py:2024-2029](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/server.py#L2024-L2029); [agent/middleware/model_call_timeout.py:25](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/middleware/model_call_timeout.py#L25); searched `rg -n -i "max_cost|cost_limit|budget_usd|spend_limit|max_spend"` in `agent` → 0 hits (No cost or spend ceiling exists; session cost is only displayed in Slack.) (verified)
  - *To reach the next level:* A token or cost cap and rate limits on side-effecting tools.
- **C L2:** The top-level loop and tool calls are bounded; sub-agents get their own model-call timeouts but no shared budget is shown. — [agent/server.py:565-577](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/server.py#L565-L577); [agent/tools/background_execute.py:29-30](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/tools/background_execute.py#L29-L30) (verified)
  - *To reach the next level:* Sub-agents and background tasks should count against one budget.
- **D L1:** Defaults are very large (5,000 model calls, 24-hour background commands). — [agent/runtime/constants.py:4-6](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/runtime/constants.py#L4-L6); [agent/tools/background_execute.py:29-30](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/tools/background_execute.py#L29-L30) (verified)
  - *To reach the next level:* Sensible defaults sized to a single task.
- **B L1:** Ceilings are hours-scale with unlimited spend, and stop leaves background sandbox processes running. — [agent/slack/stop.py:190-197](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/slack/stop.py#L190-L197); [agent/tools/background_execute.py:29-30](https://github.com/langchain-ai/open-swe/blob/8e0e13e33a94da2266e180b0ffeb7c6264832457/agent/tools/background_execute.py#L29-L30) (verified)
  - *To reach the next level:* Tight time/cost ceilings and a stop that kills in-flight work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web fetch, repo files, GitHub/Slack/Linear content, MCP results (agent/server.py:1679-1683; agent/github/comments.py:149-159) · [B] sensitive data/systems: installation-wide GitHub token via proxy, private repos, Slack (agent/github/sandbox_access.py:145-163) · [C] state change / egress: shell push, http_request, Slack posts, open sandbox egress (agent/server.py:1679-1726) · Same default session? Yes

## Highest-impact improvements
1. Make the workflow/push guard fail closed and cover every git and GitHub write path. — C2 C L0→L1, +0.075 before caps (Playbook 5)
2. Scope member-started coding sandboxes to the thread's repositories and needed permissions by default (the narrowing code already exists for public-repo events). — C1 D L1→L2, +0.050 before caps (Playbook 4)
3. Add a default egress allowlist to sandbox proxy config and withhold the thread-tools capability from sandboxed code unless enabled. — C4 B L1→L2, +0.050 before caps (Playbook 3)
4. Require the user to confirm text before save_user_instructions/save_user_skill persist it. — C6 S L1→L2, +0.075 before caps (Playbook 2)
5. Add a per-run token/cost ceiling and smaller default model-call cap. — C10 D L1→L2, +0.050 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The isolation primitive inside LangSmith sandboxes (and Daytona/E2B/Modal/Runloop) is outside this repository; C4 S credits remote execution as verified in this code, not the provider's internals.
- The ui/, desktop/ and cli/ TypeScript packages were not reviewed in depth; desktop and CLI-bridge modes run commands on the user's machine by design and are footnoted, not scored.
- Deep Agents and LangChain middleware behaviour (execute tool timeout defaults, sub-agent recursion inheritance) is inferred from how Open SWE configures them.
- No reviewer-injection text aimed at auditors was found in AGENTS.md, CLAUDE.md, README.md or .open-swe/.
