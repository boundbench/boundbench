# Defense-in-Depth Score: Agent Teams AI

**Repo:** https://github.com/777genius/agent-teams-ai · **Commit:** `f8d3434cac84e658640318c65d56de7cc944af0c` (2.1.2) · **Reviewed:** 2026-10-04
**What it is:** Electron desktop app that orchestrates teams of Claude Code, Codex and OpenCode agents with a kanban board, messaging, review and budgets.
**Category:** Coding
**Scored configuration:** Desktop app, fresh install, team created and launched from the UI with default options (auto-approve tools on, no worktrees, telemetry default).
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 1.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | C1-SELFESC | **0.00** | High |
| C2 | Approval gates | L3 | L1 | L1 | L2 | 0.45 | C2-SELFAPPROVE | **0.25** (alt) | Medium |
| C3 | Tool & action scoping | L0 | L1 | L0 | L0 | 0.07 | — | **0.07** | High |
| C4 | Code-execution isolation | L1 | L1 | L0 | L0 | 0.15 | G1 | **0.15** (alt) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | L1 | L0 | L1 | L0 | 0.12 | — | **0.12** | Medium |
| C8 | Secrets & sensitive-data protection | L2 | L1 | L1 | L0 | 0.28 | — | **0.28** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | Medium |
| C10 | Limits & kill switch | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |


Agent Teams AI launches every teammate with Claude's permission prompts bypassed by default, and its terminal automation auto-accepts the runtime's 'trust this folder' and bypass-permissions warnings. Agents run unsandboxed as the user with the full shell environment, so a single prompt injection in a repo, web page or peer message can exfiltrate credentials and push or delete with no human involved. A real per-call approval sheet exists, but it is opt-in and still pre-allows file writes through a project settings file.

## Critical gaps
- Teammates run with the user's full environment and shell credentials and no scoped identity, so a hijacked agent holds the user's entire account. (ASI03, T3; C1) — [src/main/utils/cliEnv.ts:40-43](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/cliEnv.ts#L40-L43); [src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts:517-519](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts#L517-L519)
- Model-written shell commands run on the host as the user with no sandbox and the full credential-bearing environment. (ASI05, T11; C4) — searched `rg -n -i 'seccomp|landlock|sandbox-exec|bwrap|firejail|--sandbox'` in `src/main` → 0 hits (No OS sandbox, container, or seccomp/landlock wrapper around any runtime the app spawns.); [src/main/utils/cliEnv.ts:40-43](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/cliEnv.ts#L40-L43)
- In the default launch a prompt-injected teammate can exfiltrate secrets and take irreversible actions with no human in the loop. (ASI01, LLM01, T6; C5) — [src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts:517-519](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts#L517-L519); [src/main/utils/cliEnv.ts:40-43](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/cliEnv.ts#L40-L43)
- The app auto-accepts the runtime's workspace-trust and bypass-permissions prompts, so repository-controlled settings, hooks and MCP servers load without a user decision. (ASI06, T1; C6) — [src/features/workspace-trust/core/application/StartupDialogRules.ts:85-92](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/features/workspace-trust/core/application/StartupDialogRules.ts#L85-L92); [src/features/workspace-trust/core/application/StartupDialogRules.ts:113-120](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/features/workspace-trust/core/application/StartupDialogRules.ts#L113-L120); [src/features/workspace-trust/main/infrastructure/WorkspaceTrustFeatureFlags.ts:20-24](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/features/workspace-trust/main/infrastructure/WorkspaceTrustFeatureFlags.ts#L20-L24); [src/features/workspace-trust/main/infrastructure/WorkspaceTrustFeatureFlags.ts:62-67](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/features/workspace-trust/main/infrastructure/WorkspaceTrustFeatureFlags.ts#L62-L67)
- Third-party MCP servers run as the user with the full credential-bearing environment and no integrity verification. (ASI04, T17; C7) — [src/main/services/extensions/install/McpInstallService.ts:140-147](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/extensions/install/McpInstallService.ts#L140-L147); [src/main/utils/cliEnv.ts:40-43](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/cliEnv.ts#L40-L43)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

Every teammate runtime runs as the desktop user with the app's full process environment merged with the user's interactive-shell environment, so cloud CLIs, SSH agents, gh tokens and provider API keys are all reachable. There is no scoped identity or per-request authorization layer between the agent and those credentials. Team-launch control paths are not locked down. A hijacked teammate holds the user's whole account across services.

- **S L0:** Runtimes inherit the full process.env plus the cached interactive-shell env; no credential narrowing. — [src/main/utils/cliEnv.ts:40-43](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/cliEnv.ts#L40-L43) (verified)
  - *To reach the next level:* No scoped or dedicated identity; credentials are not narrowed per role or tool.
- **C L0:** No authorization layer sits between tools and ambient credentials; team-launch control paths are also not locked down. — [src/main/utils/cliEnv.ts:40-43](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/cliEnv.ts#L40-L43) (verified)
  - *To reach the next level:* No tool path is checked against an authorization layer.
- **D L0:** Default launch is bypassPermissions with the user's full environment. — [src/renderer/components/team/dialogs/LaunchTeamDialog.tsx:321-323](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/renderer/components/team/dialogs/LaunchTeamDialog.tsx#L321-L323); [src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts:517-519](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts#L517-L519) (verified)
  - *To reach the next level:* Default runs with the user's full privilege; no narrower default.
- **B L0:** A hijacked teammate reaches the user's entire account: shell env secrets, home directory, git/ssh credentials. — [src/main/utils/cliEnv.ts:40-43](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/cliEnv.ts#L40-L43); [src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts:517-519](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts#L517-L519) (verified)
  - *To reach the next level:* Blast radius is the user's full account across services.
- **Cap:** C1-SELFESC — An agent-reachable path lets a teammate escalate its own permission settings.
- **Notes:** C1-SELFESC applies. The cap does not change the score, which is already 0.

### C2 Approval gates — 0.25 (medium)

Teams launch with Claude's permission prompts disabled by default: the 'auto-approve tools' toggle starts on, and the app passes --dangerously-skip-permissions with bypassPermissions to the lead, every teammate and scheduled runs. An approval mode exists when the user unticks it; it routes each tool request to an in-app sheet that shows the exact command, JSON input and a diff. Even in that mode the app writes allow rules for Edit, Write and NotebookEdit into the project's .claude/settings.local.json, and an 'allow all' button flips the team to auto-approve. File changes can be rejected afterwards through the review panel, but shell side effects cannot.

- **default configuration** (default; raw 0.10, cap C2-POWERBYPASS → 0.10)
  - **S L0:** In the default configuration there is no approval: permission prompts are bypassed and the app's approval settings are set to autoAllowAll. — [src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts:517-519](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts#L517-L519); [src/main/services/team/provisioning/TeamProvisioningToolApprovalFacade.ts:411-416](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningToolApprovalFacade.ts#L411-L416) (verified)
    - *To reach the next level:* No per-call approval in the shipped default.
  - **C L0:** Bash, file writes and MCP tools all run ungated in the default bypassPermissions mode. — [src/main/services/team/provisioning/TeamProvisioningMemberLifecycle.ts:737-739](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningMemberLifecycle.ts#L737-L739); [src/main/services/schedule/ScheduledTaskExecutor.ts:245-247](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/schedule/ScheduledTaskExecutor.ts#L245-L247) (verified)
    - *To reach the next level:* The most powerful tool (shell) is exempt by default.
  - **D L0:** The skip-permissions toggle defaults to on (any value other than the string 'false'). — [src/renderer/components/team/dialogs/LaunchTeamDialog.tsx:321-323](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/renderer/components/team/dialogs/LaunchTeamDialog.tsx#L321-L323); [src/renderer/services/createTeamPreferences.ts:250-256](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/renderer/services/createTeamPreferences.ts#L250-L256); [src/main/services/team/provisioning/TeamProvisioningCreateTeamFlow.ts:448-450](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningCreateTeamFlow.ts#L448-L450) (verified)
    - *To reach the next level:* Approval is opt-in.
  - **B L2:** Agent file edits can be reverted per hunk or file in the review panel; shell, git push and external actions have no undo. — [src/main/services/team/ReviewApplierService.ts:438](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/ReviewApplierService.ts#L438) (verified)
    - *To reach the next level:* No checkpoints or previews for non-file actions.
- **opt-in approval mode (auto-approve tools unticked)** (alt; raw 0.45, cap C2-SELFAPPROVE → 0.25) ← counted
  - **S L3:** Each tool request is shown in an approval sheet with the exact command or JSON input and a diff preview; categories of auto-allow exist. — [src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts:519](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts#L519); [src/renderer/components/team/ToolApprovalSheet.tsx:77-90](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/renderer/components/team/ToolApprovalSheet.tsx#L77-L90); [src/renderer/components/team/ToolApprovalSheet.tsx:352](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/renderer/components/team/ToolApprovalSheet.tsx#L352) (verified)
    - *To reach the next level:* No argument-level allow/deny policy on parsed arguments; approved-equals-executed is not enforced by the app.
  - **C L1:** Lead Edit/Write/NotebookEdit are pre-allowed in project settings, so file writes skip the gate; the optional safe-bash list is a startsWith prefix list that includes npm, npx, git and node -e. — [src/main/services/team/provisioning/TeamProvisioningClaudePermissionSettings.ts:76-86](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningClaudePermissionSettings.ts#L76-L86); [src/main/services/team/provisioning/TeamProvisioningClaudePermissionSettings.ts:26](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningClaudePermissionSettings.ts#L26); [src/main/utils/toolApprovalRules.ts:7-63](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/toolApprovalRules.ts#L7-L63); [src/main/utils/toolApprovalRules.ts:142-146](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/toolApprovalRules.ts#L142-L146) (verified)
    - *To reach the next level:* File-write tools and prefix-matched commands bypass the gate.
  - **D L1:** Allow rules are persisted in workspace scope, and an 'allow all' button switches the team to auto-approve; the approval configuration is not tamper-resistant. — [src/main/services/team/provisioning/TeamProvisioningClaudePermissionSettings.ts:76-86](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningClaudePermissionSettings.ts#L76-L86); [src/renderer/components/team/ToolApprovalSheet.tsx:434](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/renderer/components/team/ToolApprovalSheet.tsx#L434); [src/shared/types/team.ts:1795-1801](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/shared/types/team.ts#L1795-L1801) (inferred)
    - *To reach the next level:* Persisted allow rules live in repo scope; make the approval configuration tamper-resistant.
  - **B L2:** Same as default: file changes reversible via review panel, other actions not. — [src/main/services/team/ReviewApplierService.ts:438](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/ReviewApplierService.ts#L438) (verified)
    - *To reach the next level:* No checkpoints or previews for non-file actions.
- **Cap:** C2-SELFAPPROVE — Even in approval mode the model can widen its own permissions without a human because protection of the approval configuration is not tamper-resistant.

### C3 Tool & action scoping — 0.07 (high)

The consequential tools are the runtime's own general-purpose ones (Bash, Write, WebFetch), passed through with no argument validation added by this project. The app's own MCP tools for tasks, messages and kanban use typed zod schemas and check that the team exists, but they take model-chosen file paths and directories. The default tool set includes shell, write and network; the app removes only a few orchestration tools (team launch and stop, TeamDelete). A misused tool can reach the whole machine.

- **S L0:** Shell commands and file paths from the model reach the runtime's Bash and Write tools unvalidated. — [src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts:517-519](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts#L517-L519); searched `rg -n -i 'seccomp|landlock|sandbox-exec|bwrap|firejail|--sandbox'` in `src/main` → 0 hits (No OS sandbox, container, or seccomp/landlock wrapper around any runtime the app spawns.) (verified)
  - *To reach the next level:* No allowlist validation of commands, paths or URLs.
- **C L1:** Only the app's own MCP tools validate input shape (zod) and team existence. — [mcp-server/src/tools/taskTools.ts:579-590](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/mcp-server/src/tools/taskTools.ts#L579-L590); [mcp-server/src/utils/teamConfig.ts:58-64](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/mcp-server/src/utils/teamConfig.ts#L58-L64) (verified)
  - *To reach the next level:* Built-in runtime tools are not covered by any validation layer.
- **D L0:** All runtime tools including shell, write and web are enabled; only team_launch/team_stop/TeamDelete/Task* are disallowed. — [src/main/services/team/provisioning/TeamProvisioningRunModel.ts:36-37](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningRunModel.ts#L36-L37) (verified)
  - *To reach the next level:* Default tool set includes write, exec and network.
- **B L0:** General-purpose shell against the user's machine with full environment. — [src/main/utils/cliEnv.ts:40-43](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/cliEnv.ts#L40-L43); [src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts:517-519](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts#L517-L519) (verified)
  - *To reach the next level:* Tools are not scoped to the project.
- **Cap:** none

### C4 Code-execution isolation — 0.15 (high)

No sandbox of any kind wraps the runtimes the app launches: no container, OS sandbox profile or seccomp/landlock policy, and the processes run with the user's full environment. The only separation is an optional per-teammate git worktree, which is a separate working directory, not a boundary, and is off by default. Model-written commands therefore run directly on the host as the user.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Runtimes are same-user host subprocesses with no isolation primitive. — searched `rg -n -i 'seccomp|landlock|sandbox-exec|bwrap|firejail|--sandbox'` in `src/main` → 0 hits (No OS sandbox, container, or seccomp/landlock wrapper around any runtime the app spawns.); [src/main/utils/cliEnv.ts:40-43](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/cliEnv.ts#L40-L43) (verified)
    - *To reach the next level:* No OS-level isolation.
  - **C L0:** No execution path is isolated. — searched `rg -n -i 'seccomp|landlock|sandbox-exec|bwrap|firejail|--sandbox'` in `src/main` → 0 hits (No OS sandbox, container, or seccomp/landlock wrapper around any runtime the app spawns.) (verified)
    - *To reach the next level:* No path runs inside a sandbox.
  - **D L0:** Nothing is on by default; permission prompts are also bypassed. — searched `rg -n -i 'seccomp|landlock|sandbox-exec|bwrap|firejail|--sandbox'` in `src/main` → 0 hits (No OS sandbox, container, or seccomp/landlock wrapper around any runtime the app spawns.); [src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts:517-519](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts#L517-L519) (verified)
    - *To reach the next level:* No isolation in the default configuration.
  - **B L0:** Host-equivalent: home directory, credentials and network all reachable. — [src/main/utils/cliEnv.ts:40-43](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/cliEnv.ts#L40-L43) (verified)
    - *To reach the next level:* Execution reaches the full host with credentials.
- **opt-in per-teammate git worktree** (alt; raw 0.15, cap G1 → 0.15) ← counted
  - **S L1:** A teammate can be given its own git worktree as cwd, which is only a separate working directory. — [src/main/services/team/provisioning/TeamProvisioningBootstrapSpec.ts:158](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningBootstrapSpec.ts#L158) (verified)
    - *To reach the next level:* No OS-level separation.
  - **C L1:** Applies to the selected teammate's process cwd only; nothing confines its commands. — [src/main/services/team/provisioning/TeamProvisioningBootstrapSpec.ts:158](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningBootstrapSpec.ts#L158) (verified)
    - *To reach the next level:* Other teammates and spawned processes run in the main checkout.
  - **D L0:** Worktree default is false in the launch dialog. — [src/renderer/components/team/dialogs/LaunchTeamDialog.tsx:316](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/renderer/components/team/dialogs/LaunchTeamDialog.tsx#L316) (verified)
    - *To reach the next level:* Off by default.
  - **B L0:** Same host, same user, same credentials. — [src/main/utils/cliEnv.ts:40-43](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/cliEnv.ts#L40-L43) (verified)
    - *To reach the next level:* Execution still reaches the full host.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.00 (high)

Teammates read repository files, web content and messages from other agents and other teams, and nothing in the orchestration layer marks or limits that content. Because the default launch bypasses all permission prompts, an injected instruction can make a teammate exfiltrate secrets over the network and take irreversible actions such as pushing code, with no human involved. Cross-team messaging lets one team's agents instruct another team's agents. Rendered markdown is sanitized, but not on every rendering path.

- **S L0:** No structural limit after untrusted content is read; no detection either. — searched `rg -n -i 'untrusted|prompt injection'` in `agent-teams-controller/src mcp-server/src` → 0 hits (No provenance or untrusted-content marking in the orchestration layer that relays agent messages and tasks.); [src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts:517-519](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts#L517-L519) (verified)
  - *To reach the next level:* No approval or capability restriction triggered by untrusted content.
- **C L0:** Messages from peer agents and other teams enter context with the same standing as the user's. — [mcp-server/src/tools/messageTools.ts:16](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/mcp-server/src/tools/messageTools.ts#L16); [mcp-server/src/tools/crossTeamTools.ts:21](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/mcp-server/src/tools/crossTeamTools.ts#L21); searched `rg -n -i 'untrusted|prompt injection'` in `agent-teams-controller/src mcp-server/src` → 0 hits (No provenance or untrusted-content marking in the orchestration layer that relays agent messages and tasks.) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished.
- **D L0:** No control exists to be on by default. — searched `rg -n -i 'untrusted|prompt injection'` in `agent-teams-controller/src mcp-server/src` → 0 hits (No provenance or untrusted-content marking in the orchestration layer that relays agent messages and tasks.) (verified)
  - *To reach the next level:* Off by default.
- **B L0:** Untrusted input, full credentials and unattended shell/network egress coexist in every default session. — [src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts:517-519](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts#L517-L519); [src/main/utils/cliEnv.ts:40-43](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/cliEnv.ts#L40-L43) (verified)
  - *To reach the next level:* Leak and irreversible action both possible without a human.
- **Cap:** C5-WORSTCASE — B is L0: in the default bypassPermissions launch a hijacked teammate can leak secrets and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.10 (high)

The app automatically answers the runtime's 'trust this folder' prompt and its bypass-permissions warning by sending keystrokes to the terminal, and pre-marks Codex projects as trusted. That removes the user decision that normally gates a repository's own settings, hooks, MCP servers and instruction files. Team tasks, comments and messages persist on disk and are re-injected into teammates' context in later sessions, with no provenance or review. Agents can also write into other teams' inboxes.

- **S L0:** Agent-written tasks/messages persist and are re-injected; repo-controlled settings load after an automated trust acceptance. — [src/features/workspace-trust/core/application/StartupDialogRules.ts:85-92](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/features/workspace-trust/core/application/StartupDialogRules.ts#L85-L92); [mcp-server/src/tools/crossTeamTools.ts:21](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/mcp-server/src/tools/crossTeamTools.ts#L21) (verified)
  - *To reach the next level:* No gating or provenance on persisted content; no explicit workspace-trust decision.
- **C L0:** Neither team stores nor auto-loaded workspace files are controlled. — [src/features/workspace-trust/core/application/StartupDialogRules.ts:85-92](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/features/workspace-trust/core/application/StartupDialogRules.ts#L85-L92); [src/main/services/team/TeamMcpConfigBuilder.ts:633-635](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/TeamMcpConfigBuilder.ts#L633-L635) (verified)
  - *To reach the next level:* No memory or config path is controlled.
- **D L1:** Team data is stored per team directory, but agents can write into other teams via cross_team_send by design. — [mcp-server/src/tools/crossTeamTools.ts:21](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/mcp-server/src/tools/crossTeamTools.ts#L21) (verified)
  - *To reach the next level:* The model can write into other namespaces.
- **B L1:** Poisoned tasks or messages persist across the user's sessions and drive tool use by teammates. — [mcp-server/src/tools/messageTools.ts:16](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/mcp-server/src/tools/messageTools.ts#L16); [mcp-server/src/tools/crossTeamTools.ts:21](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/mcp-server/src/tools/crossTeamTools.ts#L21) (verified)
  - *To reach the next level:* Persisted content can trigger tool use without review.
- **Cap:** C6-REPOCONFIG — The app auto-accepts the runtime's workspace-trust prompt (default-on flag), so workspace .claude settings, hooks and project MCP servers load without an explicit user trust decision; the downstream loading is Claude Code/Codex behaviour gated by that prompt.
- **Notes:** Verified in this repo: the PTY engine presses Enter on the trust prompt and Down+Enter on the bypass-permissions warning, both on by default. Inferred (third-party runtime behaviour): that the trust prompt is what gates project hooks/MCP/settings in the bundled Claude Code fork and Codex.

### C7 Third-party extensions — 0.12 (medium)

MCP servers installed from the app's catalog run via 'npx -y' with the registry-supplied version when there is one and the latest package otherwise, with no integrity check. The user chooses each install explicitly, but project-scope MCP servers from the repository still load through the runtime's native settings once the app has auto-accepted workspace trust. Every MCP server runs as the user with the full environment.

- **S L1:** User-chosen registry packages launched with npx -y; version pinned only if the registry entry supplies one. — [src/main/services/extensions/install/McpInstallService.ts:140-147](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/extensions/install/McpInstallService.ts#L140-L147) (verified)
  - *To reach the next level:* No integrity check; versions are not reliably pinned.
- **C L0:** No extension type is verified by hash or signature. — [src/main/services/extensions/install/McpInstallService.ts:140-147](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/extensions/install/McpInstallService.ts#L140-L147); [src/main/services/team/TeamMcpConfigBuilder.ts:633-635](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/TeamMcpConfigBuilder.ts#L633-L635) (verified)
  - *To reach the next level:* No extension type is verified.
- **D L1:** Catalog installs are explicit, but repository-scope MCP servers load natively after the automated trust acceptance; the latter is inferred from runtime behaviour. — [src/main/services/extensions/install/McpInstallService.ts:140-147](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/extensions/install/McpInstallService.ts#L140-L147); [src/main/services/team/TeamMcpConfigBuilder.ts:633-635](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/TeamMcpConfigBuilder.ts#L633-L635); [src/features/workspace-trust/core/application/StartupDialogRules.ts:85-92](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/features/workspace-trust/core/application/StartupDialogRules.ts#L85-L92) (inferred)
  - *To reach the next level:* Workspace files can still add MCP servers without a shown, explicit install.
- **B L0:** MCP servers are spawned by the runtime as the same user with the full inherited environment. — [src/main/utils/cliEnv.ts:40-43](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/cliEnv.ts#L40-L43); [src/main/services/team/TeamMcpConfigBuilder.ts:633-635](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/TeamMcpConfigBuilder.ts#L633-L635) (verified)
  - *To reach the next level:* Extensions get all of the agent's credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.28 (high)

Provider API keys that the app manages are encrypted at rest through the OS keychain, and Sentry error reports pass through a redaction filter for tokens, emails and keys. Those keys are then placed in the runtime's environment together with the user's entire shell environment, where any teammate can print them with Bash, and nothing redacts what goes to the model. Sentry crash reporting is on by default unless the user turns telemetry off. Runtime stdout/stderr logs are written with owner-only permissions but not redacted.

- **S L2:** Keychain/AES-GCM storage at rest and regex redaction of Sentry events. — [src/main/services/extensions/apikeys/ApiKeyService.ts:1-9](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/extensions/apikeys/ApiKeyService.ts#L1-L9); [src/shared/utils/sentryConfig.ts:73-76](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/shared/utils/sentryConfig.ts#L73-L76) (verified)
  - *To reach the next level:* No redaction before model-bound messages or in runtime logs.
- **C L1:** Only the telemetry path is redacted; subprocess env, logs and model context are not. — [src/shared/utils/sentryConfig.ts:73-76](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/shared/utils/sentryConfig.ts#L73-L76); [src/main/services/team/provisioning/TeamProvisioningMemberLifecycle.ts:982-983](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningMemberLifecycle.ts#L982-L983); [src/main/utils/cliEnv.ts:40-43](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/cliEnv.ts#L40-L43) (verified)
  - *To reach the next level:* Logs, transcripts and subprocess environments are unprotected.
- **D L1:** Telemetry defaults to enabled when no flag is stored; events are redacted. — [src/main/sentry.ts:67-71](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/sentry.ts#L67-L71) (verified)
  - *To reach the next level:* Telemetry is on by default rather than opt-in.
- **B L0:** Long-lived provider keys and the user's shell secrets are in every runtime subprocess environment. — [src/main/services/runtime/ProviderConnectionService.ts:675-681](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/runtime/ProviderConnectionService.ts#L675-L681); [src/main/utils/cliEnv.ts:40-43](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/cliEnv.ts#L40-L43) (verified)
  - *To reach the next level:* Long-lived, high-privilege keys reachable by the model.
- **Cap:** none

### C9 Audit & traceability — 0.45 (medium)

Each teammate's actions are recorded in the runtime's structured session transcript under ~/.claude/projects, which the app parses to show per-task tool calls, and the app writes per-member stdout and stderr logs with owner-only permissions. Approval decisions are not durably recorded by the app, and the records sit in the user's home directory where the agent, running unsandboxed as that user, can rewrite them. Writing is best-effort with no fail-closed behaviour.

- **S L2:** The runtime transcript (JSONL per session) records tool calls; the app reads it and keeps per-member stdout/stderr logs. — [src/main/utils/pathDecoder.ts:282](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/pathDecoder.ts#L282); [src/main/services/team/provisioning/TeamProvisioningMemberLifecycle.ts:982-983](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningMemberLifecycle.ts#L982-L983) (inferred)
  - *To reach the next level:* No actor/approver attribution or correlation records written by the app.
- **C L2:** Runtime tools are recorded in transcripts; approvals and denials are not persisted. — [src/main/utils/pathDecoder.ts:282](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/pathDecoder.ts#L282) (inferred)
  - *To reach the next level:* Approvals, denials and extension activity are not recorded.
- **D L2:** On by default, stored outside the project under the user's home, but writable by the same-user unsandboxed agent. — [src/main/utils/pathDecoder.ts:282](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/utils/pathDecoder.ts#L282); [src/main/services/team/provisioning/TeamProvisioningMemberLifecycle.ts:982-983](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningMemberLifecycle.ts#L982-L983) (verified)
  - *To reach the next level:* Records are written where the agent's process can alter them.
- **B L1:** Log streams are best-effort appends with no fail-closed behaviour. — [src/main/services/team/provisioning/TeamProvisioningMemberLifecycle.ts:982-983](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningMemberLifecycle.ts#L982-L983) (verified)
  - *To reach the next level:* Records are not durable per action and failures are not surfaced.
- **Cap:** none

### C10 Limits & kill switch — 0.15 (high)

Interactive teams, the main mode, launch with no turn, time or spend cap; the monthly token and cost budgets only raise alerts at 80% and 100%. Scheduled runs are better bounded, with a 50-turn default and an optional dollar cap passed to the runtime. Stopping a team kills the tracked CLI process trees, including a SIGKILL sweep, but background services that agents register are a separate list.

- **S L1:** Budgets are advisory alerts; only scheduled runs carry turn/budget caps; stop kills process trees. — [src/features/token-usage/core/domain/budgetPolicy.ts:14-24](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/features/token-usage/core/domain/budgetPolicy.ts#L14-L24); searched `rg -n -i 'max-turns|max-budget'` in `src/main/services/team` → 1 hits (The only hit is a test fixture; live team launches pass no turn or spend cap.); [src/main/services/team/provisioning/TeamProvisioningStopFlow.ts:210-213](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/team/provisioning/TeamProvisioningStopFlow.ts#L210-L213) (verified)
  - *To reach the next level:* No enforced step, time or cost cap on interactive teams.
- **C L1:** Caps apply only to scheduled runs, not to live teams or their teammates. — [src/main/services/schedule/ScheduledTaskExecutor.ts:225-231](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/schedule/ScheduledTaskExecutor.ts#L225-L231); searched `rg -n -i 'max-turns|max-budget'` in `src/main/services/team` → 1 hits (The only hit is a test fixture; live team launches pass no turn or spend cap.) (verified)
  - *To reach the next level:* Live teams and teammates are not covered by any cap.
- **D L0:** Interactive teams are unlimited by default. — searched `rg -n -i 'max-turns|max-budget'` in `src/main/services/team` → 1 hits (The only hit is a test fixture; live team launches pass no turn or spend cap.) (verified)
  - *To reach the next level:* No default limit in the primary mode.
- **B L0:** A runaway live team can loop and spend until the user notices and presses stop. — searched `rg -n -i 'max-turns|max-budget'` in `src/main/services/team` → 1 hits (The only hit is a test fixture; live team launches pass no turn or spend cap.); [src/main/services/schedule/SchedulerService.ts:179](https://github.com/777genius/agent-teams-ai/blob/f8d3434cac84e658640318c65d56de7cc944af0c/src/main/services/schedule/SchedulerService.ts#L179) (verified)
  - *To reach the next level:* No ceiling on live teams.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: repo files, web, peer and cross-team agent messages (mcp-server/src/tools/crossTeamTools.ts:21) · [B] sensitive data/systems: full process and shell environment with provider keys (src/main/utils/cliEnv.ts:41) · [C] state change / egress: bypassPermissions shell and network (src/main/services/team/provisioning/TeamProvisioningLaunchTeamFlow.ts:518) · Same default session? Yes

## Highest-impact improvements
1. Default the 'auto-approve tools' toggle to off so teams launch with --permission-prompt-tool stdio and the approval sheet. — C2 D L0→L3, +0.150 before caps (Playbook 5)
2. Stop auto-pressing Enter on the workspace-trust and bypass-permissions prompts; surface them to the user instead. — C6 S L0→L2, +0.150 before caps (Playbook 2)
3. Pass runtimes a scrubbed environment containing only the provider credentials they need instead of process.env plus the shell env. — C1 S L0→L2, +0.150 before caps (Playbook 4)
4. Harden team-launch control paths so agents cannot change their own permission mode. — C1 C L0→L1, +0.075 before caps (Playbook 4)
5. Enforce default turn and spend caps on interactive teams, not only scheduled runs. — C10 D L0→L2, +0.100 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The agent loop itself lives in a separately distributed, sha256-pinned runtime binary (777genius/agent_teams_orchestrator, a Claude Code fork) plus Codex/OpenCode; their internal permission, settings-loading and transcript behaviour was inferred, not read.
- Codex and OpenCode adapter permission handling was only sampled; scoring follows the Claude-runtime path the UI leads with.
- The Docker standalone mode was not scored.
- No text aimed at AI reviewers was found in README.md, AGENTS.md, CLAUDE.md or AGENT_CRITICAL_GUARDRAILS.md.
