# Defense-in-Depth Score: Orca

**Repo:** https://github.com/stablyai/orca · **Commit:** `f873aaac5b2544c02f17d98a188b75dc843a3de7` · **Reviewed:** 2026-10-05
**What it is:** Agentic development environment for fleets of parallel coding agents
**Category:** Coding
**Scored configuration:** Orca desktop app, fresh install with default settings: agents launched from the composer into per-task git worktrees using Orca's default launch arguments, agent workspace trust on, plugin system off.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 1.7 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | C2-POWERBYPASS | **0.00** | High |
| C3 | Tool & action scoping | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L1 | L1 | 0.25 | C6-REPOCONFIG | **0.25** | High |
| C7 | Third-party extensions | L3 | L1 | L2 | L2 | 0.50 | G1 | **0.50** (alt) | High |
| C8 | Secrets & sensitive-data protection | L2 | L1 | L1 | L0 | 0.28 | — | **0.28** | High |
| C9 | Audit & traceability | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |


Orca launches Claude Code, Codex and the other agents it supports with their permission-skipping flags on by default, so every agent runs shell commands, edits files and uses the network without asking, as your user, with your full environment and no sandbox. It also pre-accepts each agent's folder-trust prompt, so repository-controlled agent settings load silently. The dominant risk is a prompt-injected agent acting on all your credentials unattended; switch agent permissions to Manual and turn off workspace trust before pointing Orca at repositories or issues you do not control.

## Critical gaps
- Default launch arguments switch off every supported agent's approval prompt, so shell execution runs without any human gate. (ASI09, ASI02; C2) — [src/shared/tui-agent-permissions.ts:6-11](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-permissions.ts#L6-L11); [src/shared/default-global-settings.ts:220](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/default-global-settings.ts#L220); [src/shared/tui-agent-launch-defaults.ts:131-133](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-launch-defaults.ts#L131-L133)
- In the default configuration a hijacked agent can both exfiltrate credentials and take irreversible actions with no human involved. (ASI01; C5) — [src/shared/tui-agent-permissions.ts:6-11](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-permissions.ts#L6-L11); [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29)
- Orca pre-accepts launched agents' folder-trust prompts by default, so repository files configure agent hooks, tools and settings without a trust decision. (ASI06; C6) — [src/shared/default-global-settings.ts:224](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/default-global-settings.ts#L224); [src/main/agent-workspace-trust-spawn.ts:17](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/agent-workspace-trust-spawn.ts#L17); [src/main/agent-workspace-trust-spawn.ts:73](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/agent-workspace-trust-spawn.ts#L73)
- Every agent terminal inherits the user's full environment and authority with no narrowing. (ASI03; C1) — [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29); [src/main/runtime/orca-runtime-create-terminal.ts:89](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/runtime/orca-runtime-create-terminal.ts#L89)
- Agent commands run unsandboxed as the user, with Codex's own sandbox disabled by the default flag. (ASI05; C4) — [src/shared/tui-agent-permissions.ts:6-11](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-permissions.ts#L6-L11); [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29)
- Launched agent CLIs and skills run unpinned as the user with the full environment. (ASI04; C7) — [src/main/skills/skill-update-run.ts:96](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/skills/skill-update-run.ts#L96); [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

Orca runs every agent it launches as the logged-in user and makes no attempt to narrow that authority. Each agent terminal receives Orca's full process environment, so cloud, Git and API credentials in the user's shell reach every agent and every command it runs. The local control API that agents use to drive Orca is guarded by one shared token that grants the same authority to any caller holding it. A hijacked agent therefore acts with everything the user can do.

- **S L0:** Agents run with ambient OS-user authority; terminals inherit the full process environment with no credential narrowing. — [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29); [src/main/runtime/orca-runtime-create-terminal.ts:89](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/runtime/orca-runtime-create-terminal.ts#L89) (verified)
  - *To reach the next level:* No scoped identity or per-agent credential set; agents get the user's whole environment.
- **C L0:** No authorization layer sits between agents and the user's credentials; the control API checks only a single shared token. — [src/main/runtime/runtime-rpc/runtime-rpc-request-admission.ts:128-131](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/runtime/runtime-rpc/runtime-rpc-request-admission.ts#L128-L131); [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29) (verified)
  - *To reach the next level:* No per-tool or per-agent authorization check in code.
- **D L0:** The default install launches agents with full user authority and permission-skipping flags. — [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29); [src/shared/default-global-settings.ts:220](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/default-global-settings.ts#L220) (verified)
  - *To reach the next level:* No narrower default; least privilege requires manual hardening.
- **B L0:** A hijacked agent holds the user's whole environment and shell, reaching every service the user is logged in to. — [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29); [src/shared/tui-agent-permissions.ts:6-11](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-permissions.ts#L6-L11) (verified)
  - *To reach the next level:* No containment of credentials to one project or system.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

Orca's default launch arguments switch off the approval prompts of every supported agent: Claude Code starts with --dangerously-skip-permissions, Codex with --dangerously-bypass-approvals-and-sandbox, and the other agents with their equivalent auto-approve flags. The onboarding screen shows this as a toggle that is on by default, and the code comment states that bypass is the posture a user gets until they choose otherwise. Orca adds no approval step of its own for agent actions, so shell commands, file writes, pushes and network calls run unattended. Users can pick Manual mode, which leaves approval to each agent's own prompt.

- **S L0:** No approval for agent actions in the default configuration; Orca launches agents with their approve-everything flags. — [src/shared/tui-agent-permissions.ts:6-11](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-permissions.ts#L6-L11); [src/shared/tui-agent-launch-defaults.ts:17](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-launch-defaults.ts#L17) (verified)
  - *To reach the next level:* No per-call human approval for consequential actions.
- **C L0:** The most powerful path, each agent's shell, runs without any gate by default. — [src/shared/tui-agent-permissions.ts:7](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-permissions.ts#L7); [src/shared/default-global-settings.ts:220](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/default-global-settings.ts#L220) (verified)
  - *To reach the next level:* No gate covers shell execution in the default configuration.
- **D L0:** Approval is opt-in: the shipped default and the onboarding toggle both select the permission-skipping flags. — [src/shared/default-global-settings.ts:220](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/default-global-settings.ts#L220); [src/renderer/src/components/onboarding/AgentStep.tsx:28](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/renderer/src/components/onboarding/AgentStep.tsx#L28); [src/shared/tui-agent-launch-defaults.ts:131-133](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-launch-defaults.ts#L131-L133) (verified)
  - *To reach the next level:* Approval is not on by default.
- **B L0:** Agents can push, delete and call external services with the user's credentials, with no checkpoint beyond the git worktree. — [src/shared/tui-agent-permissions.ts:6-11](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-permissions.ts#L6-L11); [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29) (verified)
  - *To reach the next level:* No rollback or dry-run for external actions; irreversible actions run unattended.
- **Cap:** C2-POWERBYPASS — The default launch arguments disable each agent's approval prompt, so shell execution, the most powerful path, runs without any gate.
- **Notes:** Manual mode (an empty arguments field) leaves approval to each third-party agent's own prompt; that control lives outside this repository and was not scored.

### C3 Tool & action scoping — 0.15 (high)

The capabilities Orca hands to agents are general-purpose. Each agent gets an unrestricted shell, and Orca's own control API lets an agent open any URL in the built-in browser, type text into any terminal pane (including other agents' panes), create worktrees and change some Orca settings. Orca's API does validate argument shapes and sizes with typed schemas, but nothing restricts hosts, paths or commands. Everything is enabled by default.

- **S L1:** Typed schemas and size limits on Orca's API, but the browser accepts any URL, terminal input is arbitrary text, and agent shells are unrestricted. — [src/shared/rpc-contract/browser-params.ts:16-18](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/rpc-contract/browser-params.ts#L16-L18); [src/main/runtime/rpc/methods/terminal/terminal-send-method.ts:24-38](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/runtime/rpc/methods/terminal/terminal-send-method.ts#L24-L38) (verified)
  - *To reach the next level:* No allowlist validation on URLs, target panes or commands.
- **C L1:** Shape validation covers Orca's own API calls only; the agent shells that do most of the work have no argument checks. — [src/main/runtime/rpc/methods/terminal/terminal-send-method.ts:24-38](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/runtime/rpc/methods/terminal/terminal-send-method.ts#L24-L38); [src/shared/tui-agent-permissions.ts:6-11](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-permissions.ts#L6-L11) (verified)
  - *To reach the next level:* Most tool paths, including agent shells, carry no validation.
- **D L0:** Exec, write and network capabilities are all enabled for every agent by default. — [src/shared/default-global-settings.ts:220](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/default-global-settings.ts#L220); [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29) (verified)
  - *To reach the next level:* No read-only or reduced default tool set.
- **B L0:** A misused agent shell reaches the whole machine and any host on the network. — [src/shared/tui-agent-permissions.ts:6-11](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-permissions.ts#L6-L11); [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29) (verified)
  - *To reach the next level:* No workspace or host scoping for agent tools.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

Agent commands run as ordinary processes of the logged-in user. Orca isolates agents only by giving each one its own git worktree directory, which is not a security boundary, and its default Codex arguments also turn off Codex's own sandbox. No OS sandbox, container or VM backend is applied to agent terminals, setup scripts or tool processes. Anything an agent runs can read the user's home directory and credentials and reach the network.

- **S L0:** Agent terminals are same-user processes; no sandbox primitive exists in the main process. — searched `rg -n -i 'sandbox-exec|bwrap|landlock|seccomp|firejail'` in `src/main` → 0 hits (no OS sandbox primitive anywhere in the main process); [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29) (verified)
  - *To reach the next level:* No OS-level or container isolation for agent execution.
- **C L0:** No execution path is sandboxed, and Codex's own sandbox is disabled by the default flag. — [src/shared/tui-agent-permissions.ts:6-11](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-permissions.ts#L6-L11); searched `rg -n -i 'sandbox-exec|bwrap|landlock|seccomp|firejail'` in `src/main` → 0 hits (no OS sandbox primitive anywhere in the main process) (verified)
  - *To reach the next level:* No path runs inside an isolation boundary.
- **D L0:** No isolation is on by default. — [src/shared/default-global-settings.ts:220](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/default-global-settings.ts#L220); searched `rg -n -i 'sandbox-exec|bwrap|landlock|seccomp|firejail'` in `src/main` → 0 hits (no OS sandbox primitive anywhere in the main process) (verified)
  - *To reach the next level:* Isolation is not available or enabled by default.
- **B L0:** Agent processes see the user's home directory, full environment and unrestricted network. — [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29); [src/main/runtime/orca-runtime-create-terminal.ts:89](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/runtime/orca-runtime-create-terminal.ts#L89) (verified)
  - *To reach the next level:* Agents are not confined to the worktree and lack network limits.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Agents in Orca read repositories, GitHub and Linear issues, web pages in the built-in browser and messages from other agents, and nothing separates that content from the user's instructions. Because agents run with approvals off, full credentials and open network access, injected instructions can leak data and take irreversible actions with no human involved. Any agent can also type into another agent's terminal through Orca's control API, so one compromised agent can steer the rest. Issue links are pasted as drafts rather than submitted, which keeps a human in the first step only.

- **S L0:** Nothing limits a hijacked agent; there is no taint tracking or approval tied to untrusted content. — [src/shared/tui-agent-permissions.ts:6-11](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-permissions.ts#L6-L11); [src/shared/default-global-settings.ts:220](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/default-global-settings.ts#L220) (verified)
  - *To reach the next level:* No capability is disabled or gated after untrusted content is read.
- **C L0:** Untrusted sources are not distinguished; peer-agent input arrives as ordinary terminal keystrokes. — [src/main/runtime/rpc/methods/terminal/terminal-send-method.ts:24-38](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/runtime/rpc/methods/terminal/terminal-send-method.ts#L24-L38); [src/renderer/src/lib/launch-work-item-direct.ts:79](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/renderer/src/lib/launch-work-item-direct.ts#L79) (verified)
  - *To reach the next level:* No source is treated as untrusted data.
- **D L0:** No untrusted-input control exists to be on by default. — [src/shared/default-global-settings.ts:220](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/default-global-settings.ts#L220); [src/shared/tui-agent-launch-defaults.ts:131-133](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-launch-defaults.ts#L131-L133) (verified)
  - *To reach the next level:* No default control to rate.
- **B L0:** A hijacked agent can exfiltrate credentials and push or delete with no human involved. — [src/shared/tui-agent-permissions.ts:6-11](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-permissions.ts#L6-L11); [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions are not gated.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.25 (high)

Orca's own repository file, orca.yaml, can define setup scripts and terminal commands, and the desktop app asks the user to approve their exact content (re-asking when it changes) before running them. However, Orca by default pre-accepts the folder-trust prompt of Claude Code, Codex, Cursor, Copilot and other agents for every workspace it opens, so repository-controlled agent configuration (instruction files, project settings, hooks and tool servers) loads in the launched agent without a trust decision. Agents can also change the default launch arguments and environment for future agents through Orca's control API. A poisoned repository configuration therefore persists and can trigger tool use in later sessions.

- **S L1:** orca.yaml commands require a content-hash approval in the desktop app, but agent workspaces are pre-trusted so repo-controlled agent config loads without a prompt. — [src/renderer/src/lib/ensure-hooks-confirmed.ts:118-123](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/renderer/src/lib/ensure-hooks-confirmed.ts#L118-L123); [src/renderer/src/lib/ensure-hooks-confirmed.ts:140](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/renderer/src/lib/ensure-hooks-confirmed.ts#L140); [src/shared/default-global-settings.ts:224](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/default-global-settings.ts#L224); [src/main/agent-workspace-trust-spawn.ts:73](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/agent-workspace-trust-spawn.ts#L73) (verified)
  - *To reach the next level:* Repository-controlled agent configuration needs an explicit workspace-trust decision.
- **C L1:** Only Orca's own orca.yaml is controlled; the pre-trusted agent config files and Orca's launch settings are not. — [src/main/agent-workspace-trust-spawn.ts:17](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/agent-workspace-trust-spawn.ts#L17); [src/shared/tui-agent-config.ts:67](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-config.ts#L67); [src/main/agent-trust-presets.ts:11-12](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/agent-trust-presets.ts#L11-L12) (verified)
  - *To reach the next level:* Auto-loaded agent instruction and settings files are not covered.
- **D L1:** Agents can rewrite the default launch arguments and environment of every agent through the control API. — [src/main/runtime/rpc/methods/client-ui.ts:22-23](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/runtime/rpc/methods/client-ui.ts#L22-L23); [src/shared/rpc-contract/client-settings-params.ts:97-101](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/rpc-contract/client-settings-params.ts#L97-L101) (verified)
  - *To reach the next level:* The model can change configuration that applies to other agents and future sessions.
- **B L1:** Repository config and changed launch settings persist across the user's sessions and can trigger tool use. — [src/shared/default-global-settings.ts:224](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/default-global-settings.ts#L224); [src/shared/rpc-contract/client-settings-params.ts:97-101](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/rpc-contract/client-settings-params.ts#L97-L101) (verified)
  - *To reach the next level:* Poisoned configuration is not limited to text output or gated actions.
- **Cap:** C6-REPOCONFIG — Orca pre-accepts the launched agents' folder-trust prompts by default (verified), so repository files can enable agent hooks, tools and settings without a user trust decision (downstream effect inferred from Orca's own note that Codex ignores .codex config without trust).

### C7 Third-party extensions — 0.50 (high)

By default Orca launches whichever agent CLIs the user has installed, unpinned, with the full user environment, and its skill updater runs npx --yes skills, which fetches the latest package each time. Because Orca pre-trusts workspaces, a repository can also add tool servers to the launched agents. Orca's own plugin system is much stronger: installed plugin content is hash-checked, reserved identities must come from the official organization, consent is re-requested when a plugin's capabilities or code expand, and plugin workers get an allowlisted environment. That plugin system is off by default.

- **default configuration** (default; raw 0.07 → 0.07)
  - **S L1:** Agent CLIs and skill updates run whatever version is current at launch, with no pinning or integrity check. — [src/main/skills/skill-update-run.ts:96](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/skills/skill-update-run.ts#L96) (verified)
    - *To reach the next level:* Versions of launched agents and skill packages are not pinned.
  - **C L0:** No default extension type is verified. — [src/main/skills/skill-update-run.ts:96](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/skills/skill-update-run.ts#L96); [src/shared/tui-agent-config.ts:67](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-config.ts#L67) (verified)
    - *To reach the next level:* No extension type launched by default is verified.
  - **D L0:** Repository files can add tool servers to launched agents because workspaces are pre-trusted. — [src/shared/default-global-settings.ts:224](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/default-global-settings.ts#L224); [src/main/agent-workspace-trust-spawn.ts:73](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/agent-workspace-trust-spawn.ts#L73) (verified)
    - *To reach the next level:* Workspace files can enable extensions without consent.
  - **B L0:** Launched agents and skills run as the user with the full environment. — [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29) (verified)
    - *To reach the next level:* Extensions are not separated from the user's credentials.
- **opt-in Orca plugin system (pluginSystemEnabled)** (alt; raw 0.50, cap G1 → 0.50) ← counted
  - **S L3:** Installed plugin trees are hash-addressed and verified, and reserved identities must resolve to the official organization. — [src/main/plugins/plugin-content-integrity.ts:29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/plugins/plugin-content-integrity.ts#L29); [src/main/plugins/plugin-install-trust.ts:24-27](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/plugins/plugin-install-trust.ts#L24-L27) (verified)
    - *To reach the next level:* No signature verification or full rug-pull re-approval on every content change was confirmed.
  - **C L1:** Covers Orca plugins only; skills and agent tool servers are outside it. — [src/main/plugins/plugin-content-integrity.ts:29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/plugins/plugin-content-integrity.ts#L29); [src/main/skills/skill-update-run.ts:96](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/skills/skill-update-run.ts#L96) (verified)
    - *To reach the next level:* Other extension types are not covered.
  - **D L2:** Plugins are off by default and enabling one records a consent fingerprint. — [src/shared/default-global-settings.ts:196](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/default-global-settings.ts#L196); [src/main/startup/main-process-plugins.ts:42](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/startup/main-process-plugins.ts#L42); [src/main/plugins/plugin-enablement.ts:11-13](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/plugins/plugin-enablement.ts#L11-L13) (verified)
    - *To reach the next level:* Consent dialog content (exact command and permissions) was not verified.
  - **B L2:** Plugin workers run in a separate process with an allowlisted environment. — [src/main/plugins/plugin-worker-env.ts:1-8](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/plugins/plugin-worker-env.ts#L1-L8) (verified)
    - *To reach the next level:* No per-plugin sandbox or scoped credentials.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C8 Secrets & sensitive-data protection — 0.28 (high)

Orca encrypts the integration tokens and cookies it stores using the operating system's secure storage, and keeps plugin workers on an allowlisted environment. Agent terminals, however, inherit Orca's full process environment, so any long-lived keys in the user's shell are visible to every agent and every command. Product telemetry is on by default for new installs; events pass a strict schema validator with enumerated fields and length caps, and crash dumps are not uploaded. Secrets are not masked in terminal output or in what agents send to their model providers.

- **S L2:** Stored integration credentials are encrypted with Electron safeStorage; no secret masking exists on terminal or model-bound paths. — [src/main/host/electron-secret-store.ts:14](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/host/electron-secret-store.ts#L14); [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29) (verified)
  - *To reach the next level:* No secret masking before logs and model-bound messages on major paths.
- **C L1:** Protection covers credentials at rest only; subprocess environments and terminal transcripts are unprotected. — [src/main/host/electron-secret-store.ts:14](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/host/electron-secret-store.ts#L14); [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29) (verified)
  - *To reach the next level:* Logs, transcripts and subprocess environments are not covered.
- **D L1:** Telemetry is opted in by default for new installs, content-free by schema. — [src/main/persistence/loading-store/loaded-cohort-migrations.ts:52-57](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/persistence/loading-store/loaded-cohort-migrations.ts#L52-L57) (verified)
  - *To reach the next level:* Telemetry is not opt-in.
- **B L0:** Long-lived keys in the user's environment reach every agent subprocess. — [src/main/providers/local-pty-spawn-environment.ts:26-29](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/providers/local-pty-spawn-environment.ts#L26-L29); [src/main/runtime/orca-runtime-create-terminal.ts:89](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/runtime/orca-runtime-create-terminal.ts#L89) (verified)
  - *To reach the next level:* Keys exposed to agents are not scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.30 (high)

Orca keeps terminal scrollback and shell history for each worktree in its application data folder, and its orchestration database records tasks, dispatches and messages between agents. It does not keep its own structured record of each tool call an agent makes; that lives in each agent's own transcript, if anywhere. The records sit outside the worktree but are writable by the same user the agents run as, and scrollback is capped and overwritten.

- **S L1:** Unstructured terminal scrollback and shell history, plus orchestration task records. — [src/main/terminal-scrollback-snapshots.ts:21-30](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/terminal-scrollback-snapshots.ts#L21-L30); [src/main/terminal-history-paths.ts:12-13](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/terminal-history-paths.ts#L12-L13) (verified)
  - *To reach the next level:* No structured per-tool-call record with arguments and results.
- **C L1:** Terminal output covers agent shells; approvals do not exist and Orca API calls are not recorded as an audit trail. — [src/main/terminal-scrollback-snapshots.ts:21-30](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/terminal-scrollback-snapshots.ts#L21-L30) (verified)
  - *To reach the next level:* Extension and API actions are not recorded.
- **D L2:** On by default in the app data folder outside the workspace, but writable by the agents' user. — [src/main/terminal-history-paths.ts:12-13](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/terminal-history-paths.ts#L12-L13); [src/main/terminal-scrollback-snapshots.ts:21-30](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/terminal-scrollback-snapshots.ts#L21-L30) (verified)
  - *To reach the next level:* Records are not written by a component the agents cannot alter.
- **B L1:** Scrollback snapshots are size-capped and written periodically, so records are best-effort. — [src/shared/terminal-scrollback-limits.ts:3](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/terminal-scrollback-limits.ts#L3) (verified)
  - *To reach the next level:* Records are not durable per action.
- **Cap:** none

### C10 Limits & kill switch — 0.20 (high)

Orca puts no step, time or spend limit on the agents it runs; those loops belong to each agent CLI. Its orchestration coordinator caps concurrent dispatches at four by default, stops nested dispatch beyond one level, and fails a task after repeated failures, but the concurrency value is taken from the caller's request. Stopping a terminal kills the whole process group, which is a real halt, while scheduled automations keep firing.

- **S L1:** Concurrency, nesting depth and a repeated-failure breaker for orchestration; no step, time or cost limit; halt kills process groups. — [src/main/runtime/orchestration/coordinator.ts:36](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/runtime/orchestration/coordinator.ts#L36); [src/shared/nested-worker-depth.ts:8](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/nested-worker-depth.ts#L8); [src/main/runtime/orchestration/coordinator-task-dispatch.ts:160](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/runtime/orchestration/coordinator-task-dispatch.ts#L160); [src/main/pty/posix-pty-process-groups.ts:256](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/pty/posix-pty-process-groups.ts#L256); searched `rg -n -i 'costLimit|spendLimit|maxCost|tokenBudget|max_cost'` in `src/main/runtime/orchestration src/main/automations` → 0 hits (no spend or token ceiling in the orchestration or automation code) (verified)
  - *To reach the next level:* No wall-clock or token/cost cap enforced in code.
- **C L1:** Limits apply only to orchestration dispatch; interactive agents and automations are unbounded. — [src/main/runtime/orchestration/coordinator.ts:36](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/main/runtime/orchestration/coordinator.ts#L36); searched `rg -n -i 'costLimit|spendLimit|maxCost|tokenBudget|max_cost'` in `src/main/runtime/orchestration src/main/automations` → 0 hits (no spend or token ceiling in the orchestration or automation code) (verified)
  - *To reach the next level:* Interactive and scheduled agent runs carry no limits.
- **D L1:** The concurrency cap is a caller-supplied parameter of the orchestration API. — [src/shared/rpc-contract/orchestration-gates-params.ts:8](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/rpc-contract/orchestration-gates-params.ts#L8) (verified)
  - *To reach the next level:* The model can raise its own concurrency limit.
- **B L0:** A runaway agent can act and spend indefinitely until a human stops it. — searched `rg -n -i 'costLimit|spendLimit|maxCost|tokenBudget|max_cost'` in `src/main/runtime/orchestration src/main/automations` → 0 hits (no spend or token ceiling in the orchestration or automation code); [src/shared/tui-agent-permissions.ts:6-11](https://github.com/stablyai/orca/blob/f873aaac5b2544c02f17d98a188b75dc843a3de7/src/shared/tui-agent-permissions.ts#L6-L11) (verified)
  - *To reach the next level:* No ceiling on run time or spend.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: repository contents, linked GitHub/Linear issues, browser pages and peer-agent terminal input (src/shared/rpc-contract/browser-params.ts:16, src/main/runtime/rpc/methods/terminal/terminal-send-method.ts:24) · [B] sensitive data/systems: full user environment and credentials in every agent terminal (src/main/providers/local-pty-spawn-environment.ts:26) · [C] state change / egress: unrestricted shell, file writes and network with approval flags on by default (src/shared/tui-agent-permissions.ts:6-11) · Same default session? Yes

## Highest-impact improvements
1. Ship empty default launch arguments (Manual mode) so each agent's own approval prompt stays on, and make the onboarding toggle default to off. — C2 D L0→L2, +0.100 before caps (Playbook 5)
2. Default agentWorkspaceTrustEnabled to false, or ask per repository before pre-trusting a workspace for launched agents. — C6 S L1→L2, +0.075 before caps (Playbook 2)
3. Remove agentDefaultArgs and agentDefaultEnv from the settings fields the shared control API accepts, so agents cannot change other agents' launch configuration. — C6 D L1→L2, +0.050 before caps (Playbook 2)
4. Clamp the orchestration maxConcurrent parameter to a server-side ceiling and add a per-dispatch wall-clock limit. — C10 D L1→L2, +0.050 before caps (Playbook 3 step 3)
5. Offer an OS sandbox profile (Seatbelt/Landlock) for agent terminals limiting writes to the worktree and network to an allowlist. — C4 S L0→L3, +0.225 before caps (Playbook 3 step 1)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The repository is very large (about two million lines); review focused on agent launch configuration, workspace trust, the runtime control API, orchestration, terminals, telemetry, secrets storage and the plugin system. The SSH relay, mobile app, cloud components and ephemeral-VM recipes were not reviewed in depth.
- The approval, sandbox and trust behaviour of the third-party agent CLIs Orca launches lives outside this repository; their effects are inferred from Orca's own code and comments.
- Worktree-creation paths reached from the CLI and mobile clients were not traced end to end for their setup-script confirmation behaviour.
- No text aimed at steering AI reviewers was found in the files read; repository instruction files (AGENTS.md, CLAUDE.md) were treated as data.
