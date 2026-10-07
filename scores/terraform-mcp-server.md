# Defense-in-Depth Score: Terraform MCP Server

**Repo:** https://github.com/hashicorp/terraform-mcp-server · **Commit:** `1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81` · **Reviewed:** 2026-10-03
**What it is:** HashiCorp MCP server for Terraform Registry and HCP Terraform workspace operations
**Category:** Infrastructure & Ops
**Scored configuration:** Local stdio server (no subcommand, as in the README's docker run -i examples) with TFE_TOKEN set, default --toolsets=all, ENABLE_TF_OPERATIONS unset, info-level logging, telemetry off.
**Agent surface (default):** code execution yes · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 4.3 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L1 | L0 | 0.12 | C1-SELFESC | **0.12** | High |
| C2 | Approval gates | L1 | L1 | L2 | L0 | 0.25 | C2-POWERBYPASS | **0.25** | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C4 | Code-execution isolation | L2 | L1 | L1 | L0 | 0.28 | G2 | **0.25** | Medium |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L0 | 0.07 | C5-WORSTCASE | **0.07** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C10 | Limits & kill switch | L2 | L2 | L3 | L1 | 0.50 | — | **0.50** | High |

Controls where a risk surface exists: 2.27 / 8.0 (28%); 2 criteria scored SA (surface absent).

Despite the documentation saying only registry tools are on by default, the code enables every tool group, so giving the server a Terraform token hands the model write access to workspaces, variables, teams and access grants across every organization that token reaches. Delete and apply tools sit behind ENABLE_TF_OPERATIONS, though enforcement of that gate does not cover every path, and the model can point a workspace at any repository and run its code with attached credentials. Handling of sensitive values on the logging path is also not locked down. Run it with --toolsets=registry or a read-only, narrowly scoped team token.

## Critical gaps
- The default tool set exposes grant_team_access and add_team_member, letting a hijacked session raise its own team's access with the user's org-wide token. (ASI03, T3; C1) — [pkg/cli/commands.go:121](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/cli/commands.go#L121); [pkg/tools/tfe/grant_team_access.go:54](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/grant_team_access.go#L54); [pkg/client/tfe_client.go:209-216](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/tfe_client.go#L209-L216)
- The model can create a workspace from any VCS repository, attach credential-bearing variable sets and queue a plan, running chosen code with those credentials, and can switch execution off the remote runners. (ASI05, T11; C4) — [pkg/tools/tfe/create_workspace.go:160-174](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/create_workspace.go#L160-L174); [pkg/tools/tfe/variable_sets.go:275](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/variable_sets.go#L275); [pkg/tools/tfe/update_workspace.go:138-145](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/update_workspace.go#L138-L145)
- Untrusted registry content reaches the model in the same session as sensitive reads and irreversible writes, with no server-side separation. (ASI01, LLM01, T6; C5) — [pkg/tools/registry/get_provider_details.go:65](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/registry/get_provider_details.go#L65); [pkg/toolsets/registry.go:89](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/toolsets/registry.go#L89)

## Criterion details

### C1 Identity & least privilege — 0.12 (high)

The server acts with one long-lived Terraform API token taken from TFE_TOKEN or, if that is unset, silently from the user's Terraform CLI credentials file. Every tool uses that same token for reads and writes, and in the default stdio mode there is no server-side authorization layer: whatever the token can do, the model can do. The token is typically a user token, so the agent holds the user's full HCP Terraform authority across organizations, and tools such as grant_team_access (including 'admin') and add_team_member let it widen access for its own team.

- **S L0:** A single ambient token (env var or ~/.terraform.d/credentials.tfrc.json) is used for all operations, and team tools let the agent grant its own team admin access. — [pkg/client/tfe_client.go:209-216](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/tfe_client.go#L209-L216); [pkg/client/credentials.go:40](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/credentials.go#L40); [pkg/client/tfe_client.go:83-86](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/tfe_client.go#L83-L86); [pkg/tools/tfe/grant_team_access.go:54](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/grant_team_access.go#L54) (verified)
  - *To reach the next level:* Use separate read and write tokens or a narrowly scoped team token, and remove or gate self-escalation tools.
- **C L1:** All tools obtain the same per-session go-tfe client, so every call is checked only by HCP Terraform's RBAC on that token; no server-side check exists in stdio mode. — [pkg/client/tfe_client.go:157-163](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/tfe_client.go#L157-L163); searched `rg -n 'rganizationAllowlist'` in `pkg/mcp-mark3labs/stdio.go` → 0 hits (The organization allowlist (the only server-side authorization layer) is wired only into the streamable-HTTP server, not stdio.) (verified)
  - *To reach the next level:* Route every tool call through a server-side authorization layer (e.g., an org/workspace allowlist) in all transports, failing closed.
- **D L1:** Destructive tools are off unless ENABLE_TF_OPERATIONS=true, but the default --toolsets value is 'all', so supplying a token alone exposes write, team and permission tools. — [pkg/cli/commands.go:121](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/cli/commands.go#L121); [pkg/tools/dynamic_tool.go:109](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/dynamic_tool.go#L109) (verified)
  - *To reach the next level:* Default to a read-only tool set and require explicit operator elevation for any write tool.
- **B L0:** A hijacked session holds the user's token: org-wide writes to workspaces, variables, teams and access grants across every organization the token reaches. — [pkg/toolsets/registry.go:53-55](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/toolsets/registry.go#L53-L55); [pkg/toolsets/registry.go:89](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/toolsets/registry.go#L89) (verified)
  - *To reach the next level:* Narrow the token's reach (single org/project, mostly read) so a compromise is confined to one tenant.
- **Cap:** C1-SELFESC — grant_team_access and add_team_member are enabled by default with a token and let the agent raise its own team's access (up to workspace/project admin) or add members to privileged teams.
- **Notes:** Streamable-HTTP mode (opt-in) passes each client's bearer token through to HCP Terraform (C1-PASSTHRU pattern); its client authentication is not locked down.

### C2 Approval gates — 0.25 (high)

The server does not ask for approval itself; it relies on the host. It tags tools with read-only/destructive hints, and the deletes, force-unlock, run apply/discard and destroy runs are not registered unless the operator sets ENABLE_TF_OPERATIONS=true. But several powerful tools are tagged non-destructive (update_workspace, grant_team_access, add_team_member), there is no dry-run or read-only mode, and enforcement of the gate does not cover every path. The only 'confirm first' rule is a sentence in the server instructions.

- **S L1:** Annotations exist on most tools, but update_workspace (described in its own text as 'potentially destructive'), grant_team_access and add_team_member are marked destructiveHint=false. — [pkg/tools/tfe/update_workspace.go:24-28](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/update_workspace.go#L24-L28); [pkg/tools/tfe/grant_team_access.go:30](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/grant_team_access.go#L30); [pkg/tools/tfe/add_team_member.go:27](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/add_team_member.go#L27); searched `rg -n -g '!*_test.go' 'DestructiveHintAnnotation'` in `pkg/tools` → 50 hits (12 tool definitions set no explicit destructive hint (variable, variable-set, tag and policy-set tools); mcp-go v0.58.0 NewTool then defaults destructiveHint=true (library behaviour), which over-warns rather than under-warns.); searched `rg -n -g '!*_test.go' 'mcp.NewTool\('` in `pkg/tools` → 62 hits (62 tool definitions (create_run has two variants).) (verified)
  - *To reach the next level:* Give every mutating tool an accurate destructive hint and separate configuration-changing tools from safe ones.
- **C L1:** Only tools flagged RequiresTFOps are withheld, and enforcement of that gate does not cover every path. — [pkg/tools/dynamic_tool.go:109](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/dynamic_tool.go#L109) (verified)
  - *To reach the next level:* Make every path to an apply cross the gate, and gate execution-mode changes.
- **D L2:** The destructive-tool gate is on by default but a single env var (ENABLE_TF_OPERATIONS) turns it off with no warning; the confirmation rule in the instructions is prompt-only. — [pkg/tools/dynamic_tool.go:95](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/dynamic_tool.go#L95); [pkg/instructions/instructions.md:13](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/instructions/instructions.md#L13) (verified)
  - *To reach the next level:* Require an explicit, loudly named operator flag and log/warn when destructive tools are enabled.
- **B L0:** A wrongly approved or bypassing call can delete variables in shared variable sets, grant admin access, or apply irreversible infrastructure changes, with no preview or rollback from the server. — [pkg/toolsets/registry.go:89](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/toolsets/registry.go#L89); searched `rg -n -g '!*_test.go' 'dry_run|dryRun|DryRun|READ_ONLY|ReadOnlyMode'` in `pkg cmd` → 0 hits (No dry-run parameter or read-only mode in shipped code (the only hits are in test files, excluded).) (verified)
  - *To reach the next level:* Offer plan previews/dry-runs for every state change and bound how much a session can change.
- **Cap:** C2-POWERBYPASS — The most powerful action, applying infrastructure changes, is reachable in the default configuration without reliably crossing the ENABLE_TF_OPERATIONS gate.

### C3 Tool & action scoping — 0.45 (high)

Tools are narrow, purpose-built wrappers over the HCP Terraform API (no shell, no generic HTTP), with typed parameters and some value checks (access levels, execution modes). But schema enums and page-size limits are advertised to the model rather than enforced, VCS repository identifiers are unrestricted, and the default enables every tool group once a token is present, so a misused tool reaches every workspace, team and variable set the token can touch.

- **S L2:** Typed parameters with some in-code checks (team access levels, execution_mode switch), but run_type enums and pageSize max are schema-only and repository identifiers are unchecked. — [pkg/tools/tfe/grant_team_access.go:54](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/grant_team_access.go#L54); [pkg/tools/tfe/update_workspace.go:138-145](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/update_workspace.go#L138-L145); [pkg/utils/pagination.go:90-94](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/utils/pagination.go#L90-L94); [pkg/tools/tfe/create_workspace.go:160-174](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/create_workspace.go#L160-L174) (verified)
  - *To reach the next level:* Enforce allowlists and bounds in code for every argument (run types, repositories, org/workspace scope).
- **C L2:** Most tools validate required strings and some values; nothing validates centrally. — [pkg/tools/tfe/create_run.go:66](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/create_run.go#L66) (verified)
  - *To reach the next level:* Validate every tool's arguments in one shared layer.
- **D L2:** Tool groups and individual tools are selectable, but the default --toolsets is 'all', which includes write tools (destructive ones need ENABLE_TF_OPERATIONS). — [pkg/cli/commands.go:121](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/cli/commands.go#L121); [README.md:643](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/README.md#L643) (verified)
  - *To reach the next level:* Ship a read-only default (the registry group the docs describe) and require explicit enabling of write tools.
- **B L1:** Misuse reaches every organization and workspace the token can access, with only API page-size limits. — [pkg/tools/tfe/variable_sets.go:275](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/variable_sets.go#L275) (verified)
  - *To reach the next level:* Scope tools to configured organizations/workspaces and bound quantities per call.
- **Cap:** none

### C4 Code-execution isolation — 0.25 (medium)

The server runs nothing locally, but its purpose is to trigger Terraform runs, and a Terraform plan executes provider and data-source code on HCP Terraform workers. The model can create a workspace backed by any repository the org's VCS connection can read, attach variable sets (often cloud credentials) and queue a plan, all without the destructive-operations switch. How those workers are isolated is outside this repo, and the model can also switch a workspace to self-hosted agent or local execution.

- **S L2:** Execution happens on HCP Terraform's remote run environment, not in the server process; its isolation is provider-side and cannot be verified from this repo. — searched `rg -n -g '!*_test.go' 'exec\.Command|os/exec|plugin\.Open|go-plugin'` in `pkg cmd go.mod` → 0 hits (No local process execution or plugin loading anywhere in the server.); [pkg/tools/tfe/create_run.go:101](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/create_run.go#L101) (inferred)
  - *To reach the next level:* Isolation the server can rely on and document (e.g., refusing execution modes other than remote).
- **C L1:** Remote runs go through HCP Terraform's workers, but the model can move a workspace to 'agent' execution (self-hosted agents) or 'local'. — [pkg/tools/tfe/update_workspace.go:138-145](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/update_workspace.go#L138-L145) (inferred)
  - *To reach the next level:* Keep every model-triggered run on the provider's remote workers and refuse execution-mode changes.
- **D L1:** update_workspace lets the model change execution_mode with no human step or operator flag. — [pkg/tools/tfe/update_workspace.go:138-145](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/update_workspace.go#L138-L145); [pkg/tools/tfe/update_workspace.go:52](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/update_workspace.go#L52) (verified)
  - *To reach the next level:* Make execution-mode changes an operator-only setting or gate them behind ENABLE_TF_OPERATIONS.
- **B L0:** Code in a model-chosen repository runs with whatever variable sets the model attaches, which commonly hold cloud provider credentials, plus full network egress from the worker. — [pkg/tools/tfe/create_workspace.go:160-174](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/create_workspace.go#L160-L174); [pkg/tools/tfe/variable_sets.go:275](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/variable_sets.go#L275) (verified)
  - *To reach the next level:* Prevent the model from attaching credential-bearing variable sets or choosing arbitrary repositories.
- **Cap:** G2 — The model can bypass the remote-runner boundary at runtime by setting a workspace's execution_mode to 'agent' or 'local' via update_workspace (pkg/tools/tfe/update_workspace.go:138-145).

### C5 Untrusted input blast radius — 0.07 (high)

The server returns raw third-party content (public registry provider and module documentation anyone can publish, plan and apply logs, run comments) as plain text with no marking of where it came from, and offers no read-only or no-egress mode. In the default configuration a hijacked host model can read secrets-bearing data, change access, delete variables and push changes to infrastructure through the same session, with nothing in the server separating untrusted input from those actions.

- **S L1:** Outputs are plain text (raw registry markdown or JSON:API payloads) with no provenance; tool descriptions carry only static tool-sequencing hints. — [pkg/tools/registry/get_provider_details.go:65](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/registry/get_provider_details.go#L65); searched `rg -n -g '!*_test.go' 'untrusted|provenance|redact|Redact'` in `pkg cmd` → 0 hits (No provenance tagging of returned content and no redaction helpers.) (verified)
  - *To reach the next level:* Return structured content that separates third-party text from metadata and flag it as untrusted.
- **C L0:** No source of untrusted content (registry docs, logs, run comments) is distinguished from any other output. — searched `rg -n -g '!*_test.go' 'untrusted|provenance|redact|Redact'` in `pkg cmd` → 0 hits (No provenance tagging of returned content and no redaction helpers.) (verified)
  - *To reach the next level:* Mark every third-party content source the same way.
- **D L0:** There is no provenance or Rule-of-Two mode to turn on. — searched `rg -n -g '!*_test.go' 'dry_run|dryRun|DryRun|READ_ONLY|ReadOnlyMode'` in `pkg cmd` → 0 hits (No dry-run parameter or read-only mode in shipped code (the only hits are in test files, excluded).) (verified)
  - *To reach the next level:* Ship a default read-only mode that drops the state-change leg.
- **B L0:** A hijacked session can both leak (plan JSON output, variable values, code run on workers) and make irreversible changes (variable deletion, access grants, triggered runs) unattended by the server. — [pkg/tools/tfe/get_plan_json_output.go:45](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/get_plan_json_output.go#L45); [pkg/toolsets/registry.go:89](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/toolsets/registry.go#L89) (verified)
  - *To reach the next level:* Drop one leg by default: no write tools when untrusted registry content is served, or no sensitive reads.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 1.00 (high)

The server keeps no memory, vector store or persistent context, and loads no configuration from the working directory: settings come from environment variables and flags, credentials from the user-scope Terraform CLI file, and its instructions are compiled into the binary. Nothing the model reads can persist into later sessions through the server.

- **Structural absence:** searched `rg -n -g '!*_test.go' 'godotenv|dotenv|ReadInConfig|SetConfigFile|AGENTS\.md|os\.WriteFile'` in `pkg cmd` → 0 hits (No dotenv loading, config-file discovery, instruction-file loading or file writes (viper only reads env via AutomaticEnv).); [pkg/instructions/instructions.go:9-12](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/instructions/instructions.go#L9-L12)

### C7 Third-party extensions — 1.00 (high)

The server loads no plugins, launches no other MCP servers and executes no downloaded code in its own process. Third-party Terraform providers and modules are only fetched and run by HCP Terraform workers, which is covered under code execution.

- **Structural absence:** searched `rg -n -g '!*_test.go' 'exec\.Command|os/exec|plugin\.Open|go-plugin'` in `pkg cmd go.mod` → 0 hits (No local process execution or plugin loading anywhere in the server.)

### C8 Secrets & sensitive-data protection — 0.25 (high)

The API token is read from the environment or the user's Terraform credentials file, never returned to the model, and only a hash of it is kept for cache comparison. But sensitive values on the server's logging path are not fully protected. Plan JSON output, which can contain sensitive values, is returned to the model unredacted. Telemetry is off by default and content-free.

- **S L1:** Token comes from env/credentials file and is not logged, but there is no redaction of tool arguments or results. — [pkg/client/tfe_client.go:209-216](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/tfe_client.go#L209-L216); searched `rg -n -g '!*_test.go' 'untrusted|provenance|redact|Redact'` in `pkg cmd` → 0 hits (No provenance tagging of returned content and no redaction helpers.) (verified)
  - *To reach the next level:* Redact sensitive fields in logs and model-bound results on all main paths.
- **C L1:** Only the token's own handling is protected; logs, model-bound results and errors are not filtered. — [pkg/tools/tfe/workspace_variables.go:121](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/workspace_variables.go#L121) (verified)
  - *To reach the next level:* Protect logs and transcripts as well as the token.
- **D L1:** Telemetry (OTEL, Instana) is opt-in and content-free, but default logging does not fully protect sensitive values. — [pkg/otelmetrics/otelmetrics.go:26](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/otelmetrics/otelmetrics.go#L26); [pkg/instana/instana.go:17](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/instana/instana.go#L17) (verified)
  - *To reach the next level:* Make default logging content-free or redacted.
- **B L1:** A leak exposes a long-lived HCP Terraform token or sensitive workspace values, usually scoped to a user's organizations. — [pkg/client/tfe_client.go:83-86](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/tfe_client.go#L83-L86); [pkg/tools/tfe/get_plan_json_output.go:45](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/tools/tfe/get_plan_json_output.go#L45) (verified)
  - *To reach the next level:* Use short-lived, narrowly scoped tokens.
- **Cap:** none
- **Notes:** Plan JSON output containing unredacted sensitive values is HCP Terraform API behaviour (json-output vs json-output-redacted), not verified in this repo.

### C9 Audit & traceability — 0.38 (high)

Every tool call is written to the server log with its name and arguments before it runs, through a middleware registered for all tools. The record is plain text by default, has no result status or caller identity in stdio mode, and goes to stderr where the host decides what to keep. HCP Terraform keeps its own audit trail, which is independent of this server.

- **S L1:** An unstructured logrus line per tool call (name and arguments) with timestamp; no result status or actor fields. — [pkg/client/tool_logging_middleware.go:18](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/tool_logging_middleware.go#L18); [pkg/logging/logging.go:103-117](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/logging/logging.go#L103-L117) (verified)
  - *To reach the next level:* Write a structured record per call with result status and requesting principal.
- **C L2:** The logging middleware wraps every registered tool. — [pkg/mcp-mark3labs/server.go:28-29](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/mcp-mark3labs/server.go#L28-L29) (verified)
  - *To reach the next level:* Also record approvals/denials and authorization decisions.
- **D L2:** On by default and written by the server to stderr/log file, outside anything the tools can touch, but configurable by env. — [pkg/logging/logging.go:103-117](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/logging/logging.go#L103-L117) (verified)
  - *To reach the next level:* Write to a sink the server process cannot alter.
- **B L1:** The line is written synchronously before the call, but write failures are ignored and actions proceed. — [pkg/client/tool_logging_middleware.go:18](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/tool_logging_middleware.go#L18) (verified)
  - *To reach the next level:* Surface logging failures and flush durably per action.
- **Cap:** none

### C10 Limits & kill switch — 0.50 (high)

All tool calls pass a rate limiter (10 per second globally, 5 per second per session by default), and HTTP requests to the registry and HCP Terraform have a 10-second timeout with three retries. But registry pagination loops have no page limit, registry requests are not tied to the caller's cancellation, and runs the server starts keep executing on HCP Terraform after the session ends.

- **S L2:** Server-enforced per-request timeouts and global/per-session rate limits; no cap on paginated loops and no cancellation of in-flight registry calls. — [pkg/client/ratelimit.go:32-34](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/ratelimit.go#L32-L34); [pkg/client/registry.go:37-39](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/registry.go#L37-L39); [pkg/client/registry.go:107-111](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/registry.go#L107-L111); [pkg/client/registry.go:77](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/registry.go#L77) (verified)
  - *To reach the next level:* Cap every operation (page counts, output size) and propagate cancellation.
- **C L2:** The rate limiter wraps every tool; timeouts cover every outbound HTTP call; remote runs are not bounded. — [pkg/mcp-mark3labs/server.go:28](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/mcp-mark3labs/server.go#L28); [pkg/client/registry.go:37-39](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/registry.go#L37-L39) (verified)
  - *To reach the next level:* Bound or cancel remote work the server starts.
- **D L3:** Sensible rate-limit defaults that only the operator can change via environment variables. — [pkg/client/ratelimit.go:40-46](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/ratelimit.go#L40-L46); [pkg/client/ratelimit.go:32-34](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/ratelimit.go#L32-L34) (verified)
  - *To reach the next level:* Enforce hard ceilings that configuration cannot exceed.
- **B L1:** At 5 calls per second a runaway session can make hundreds of changes a minute, and triggered runs continue after stop. — [pkg/client/ratelimit.go:32-34](https://github.com/hashicorp/terraform-mcp-server/blob/1d9a2e03a933e0fd00caffa1d4b8c18f9cec0a81/pkg/client/ratelimit.go#L32-L34) (verified)
  - *To reach the next level:* Tighter per-session ceilings on mutating calls and cancellation of runs on stop.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Public registry docs and module READMEs returned raw (pkg/tools/registry/get_provider_details.go:65); plan/apply logs and run comments. · [B] sensitive data/systems: HCP Terraform token (pkg/client/tfe_client.go:211-214), workspace variables and plan JSON output (pkg/tools/tfe/get_plan_json_output.go:45). · [C] state change / egress: update_workspace, create_run, variable and access tools; remote runs with network egress. · Same default session? Yes

## Highest-impact improvements
1. Change the --toolsets default from 'all' to 'default' (registry only), matching the README and CHANGELOG. — C3 D L2→L3, +0.050 before caps (Playbook 3 step 1)
2. Mark update_workspace, grant_team_access, add_team_member and create_team destructive, and set explicit hints on every tool. — C2 S L1→L2, +0.075 before caps (Playbook 5)
3. Route every path that can lead to an applied run through ENABLE_TF_OPERATIONS, and move execution_mode and VCS-repo changes behind it. — C2 C L1→L2, +0.075 before caps (Playbook 5)
4. Add redaction of sensitive values on all logging paths. — C8 S L1→L2, +0.075 before caps (Playbook 4)
5. Stop the model from changing execution_mode so runs always stay on remote workers. — C4 D L1→L2, +0.050 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Pinned commit is 35 commits after tag v1.3.0 and is not itself tagged, so version is null.
- Scored the default stdio mode. Streamable-HTTP mode (opt-in) was reviewed for footnotes only: it passes client tokens through; its client authentication and TLS handling are not locked down.
- The experimental official go-sdk server (TF_X_OFFICIAL_SDK_ENABLED, HTTP mode only) was not scored in detail.
- Several downstream effects are HCP Terraform/go-tfe behaviour inferred from library docs, not verified here: plan json-output containing unredacted sensitive values, and the isolation of HCP Terraform remote workers.
- No attempts to steer AI reviewers were found in the repository.
