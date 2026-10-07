# Defense-in-Depth Score: Symphony

**Repo:** https://github.com/openai/symphony · **Commit:** `be10a1b79df723d6d7612b5651c8522704dafb2e` · **Reviewed:** 2026-10-05
**What it is:** Turns project work into isolated autonomous Codex implementation runs
**Category:** Coding
**Scored configuration:** Elixir reference implementation started as the README describes (`./bin/symphony ./WORKFLOW.md`) with the shipped elixir/WORKFLOW.md and a Linear personal API key (approval policy never, workspace-write sandbox with network on, full environment inherited).
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 1.3 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L1 | 0.05 | C2-POWERBYPASS | **0.05** | High |
| C3 | Tool & action scoping | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C4 | Code-execution isolation | L2 | L1 | L2 | L0 | 0.33 | — | **0.33** | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L0 | 0.05 | C6-REPOCONFIG | **0.05** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | C7-RCELOAD | **0.00** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C9 | Audit & traceability | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C10 | Limits & kill switch | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | High |


Symphony polls a tracker and runs one fully autonomous Codex agent per ticket with no human approval anywhere: the shipped workflow auto-accepts approval requests, enables network inside the sandbox, and gives the agent a raw Linear API tool using your personal key. Codex's sandbox keeps writes inside each ticket's workspace, but commands can read your whole disk and environment and send data anywhere, and setup and cleanup hooks run on the host. The dominant risk is prompt injection through ticket text or PR comments leading to credential leaks and irreversible pushes or tracker changes. The project calls itself an engineering preview for trusted environments, and that is the only safe way to run it.

## Critical gaps
- The agent acts with the operator's full Linear account and can reach the operator's environment and home-directory credentials with network on, with no authorization layer in code. (ASI03, T3; C1) — [elixir/lib/symphony_elixir/linear/agent_tool.ex:8-11](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/linear/agent_tool.ex#L8-L11); [elixir/WORKFLOW.md:34](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L34)
- No approval gate covers shell commands or tracker mutations; under the shipped workflow approval requests are accepted automatically. (ASI09, ASI02, T10; C2) — [elixir/lib/symphony_elixir/codex/app_server.ex:55](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/codex/app_server.ex#L55); [elixir/WORKFLOW.md:35](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L35)
- Sandboxed commands can read the whole disk and the operator's full environment with unrestricted network in the shipped workflow, and workspace hooks run on the host. (ASI05, T11; C4) — [elixir/lib/symphony_elixir/config/schema.ex:570-579](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/config/schema.ex#L570-L579); [elixir/WORKFLOW.md:36-39](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L36-L39)
- Prompt injection via ticket text or PR comments can lead to both data exfiltration and irreversible actions with no human involved. (ASI01, T6, LLM01; C5) — [elixir/WORKFLOW.md:36-39](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L36-L39); [elixir/WORKFLOW.md:180](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L180)
- The shipped workflow auto-trusts the workspace repository's tool configuration, which then configures host-side hook commands. (ASI06, T1; C6) — [elixir/WORKFLOW.md:23-26](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L23-L26); [elixir/lib/symphony_elixir/workspace.ex:397-405](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/workspace.ex#L397-L405)
- Model-chosen package installs run unattended with network on, as the operator with the full environment. (ASI04, T17; C7) — [elixir/WORKFLOW.md:36-39](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L36-L39); [elixir/WORKFLOW.md:34](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L34)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

Symphony acts with the operator's own authority. The Linear tool it gives the agent sends any GraphQL query or mutation using the operator's personal Linear API key, so the agent can read or change anything that user can in the whole Linear workspace; the only limit is the prompt telling it to stay on its ticket. The shipped workflow also tells Codex to pass the operator's full environment to every command, and the default sandbox policy can read the whole disk, so cloud, Git and SSH credentials are within reach. The one narrowing step is that the Linear key itself is removed from the Codex process environment.

- **S L0:** The agent's tracker tool uses one operator-held personal API key for any query or mutation, and Codex runs with the operator's ambient credentials. — [elixir/lib/symphony_elixir/linear/agent_tool.ex:8-11](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/linear/agent_tool.ex#L8-L11); [elixir/lib/symphony_elixir/config/schema.ex:411-412](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/config/schema.ex#L411-L412) (verified)
  - *To reach the next level:* No dedicated or scoped identity; reads and writes share the operator's full-access personal key.
- **C L0:** No authorization check exists in code on tool calls; staying on the current ticket and repository is requested only in the prompt. — [elixir/WORKFLOW.md:73](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L73); [elixir/WORKFLOW.md:34](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L34) (verified)
  - *To reach the next level:* No code-level authorization layer on the tracker tool or on commands; Codex children get the full operator environment.
- **D L0:** The shipped workflow runs Codex with the operator's full environment inherited and the tracker tool bound to the operator's personal key. — [elixir/WORKFLOW.md:34](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L34); [elixir/lib/symphony_elixir/config/schema.ex:411-412](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/config/schema.ex#L411-L412) (verified)
  - *To reach the next level:* Least privilege requires the operator to issue a restricted key and remove the environment-inherit setting themselves.
- **B L0:** A hijacked agent holds the operator's whole Linear account and can read the operator's home directory and environment credentials with network access on. — [elixir/lib/symphony_elixir/linear/agent_tool.ex:8-11](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/linear/agent_tool.ex#L8-L11); [elixir/lib/symphony_elixir/config/schema.ex:570-579](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/config/schema.ex#L570-L579); [elixir/WORKFLOW.md:36-39](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L36-L39) (verified)
  - *To reach the next level:* Credentials reach the whole tracker workspace and every service the operator's environment and home directory hold.
- **Cap:** none

### C2 Approval gates — 0.05 (high)

Symphony is built to run without a human in the loop. With the shipped workflow (approval policy 'never') any approval request Codex raises is answered 'accept for session' automatically, including MCP tool-call approvals; with the built-in default policy, escalations are instead rejected and the ticket is parked as blocked, but no human is ever shown the exact action to approve. Shell commands, pushes, PR comments and every tracker mutation run with no gate. The 'Human Review' and 'Merging' ticket states are a workflow convention in the prompt, and the agent holds a tool that can change ticket state itself.

- **S L0:** Approval requests are auto-accepted for the session under the shipped policy; no path presents a call to a human for a decision. — [elixir/lib/symphony_elixir/codex/app_server.ex:55](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/codex/app_server.ex#L55); [elixir/lib/symphony_elixir/codex/app_server.ex:761-781](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/codex/app_server.ex#L761-L781); [elixir/WORKFLOW.md:35](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L35) (verified)
  - *To reach the next level:* No per-call human approval of any kind exists.
- **C L0:** Shell commands inside the sandbox and Symphony-executed tracker tool calls never reach any gate. — [elixir/lib/symphony_elixir/codex/app_server.ex:587-620](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/codex/app_server.ex#L587-L620); [elixir/WORKFLOW.md:35](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L35) (verified)
  - *To reach the next level:* The most powerful paths (shell, tracker mutations) are not gated.
- **D L0:** There is no human-approval mode to turn on; the shipped workflow sets the auto-approving policy. — [elixir/WORKFLOW.md:35](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L35); [elixir/lib/symphony_elixir/config/schema.ex:183-191](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/config/schema.ex#L183-L191) (verified)
  - *To reach the next level:* No approval gate exists to be on by default.
- **B L1:** Workspace changes are git-reversible, but pushes, PR comments, merges via the land skill and tracker mutations are external and mostly irreversible. — [elixir/WORKFLOW.md:114](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L114); [elixir/lib/symphony_elixir/linear/agent_tool.ex:8-11](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/linear/agent_tool.ex#L8-L11) (verified)
  - *To reach the next level:* No previews, checkpoints or bounds on external actions such as tracker mutations, pushes and comments.
- **Cap:** C2-POWERBYPASS — Shell execution and the tracker mutation tool run with no approval gate in the shipped configuration.

### C3 Tool & action scoping — 0.05 (high)

The agent's tools are general-purpose: Codex's shell plus a Linear tool that forwards any GraphQL document and variables unchanged, checking only that a query string is present. Nothing restricts which issues, projects or operations the tracker tool touches, and the shipped workflow enables shell, file writes, network and the tracker tool all at once. The reach of shell commands is narrowed only by the Codex sandbox, which limits writes to the workspace but allows reading the whole disk.

- **S L0:** The tracker tool is a raw GraphQL passthrough; argument handling only checks that a non-empty query and an object of variables were supplied. — [elixir/lib/symphony_elixir/linear/agent_tool.ex:12-27](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/linear/agent_tool.ex#L12-L27); [elixir/lib/symphony_elixir/linear/agent_tool.ex:94-112](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/linear/agent_tool.ex#L94-L112) (verified)
  - *To reach the next level:* No allowlist of operations, fields or target issues for tracker calls.
- **C L0:** No tool in the scored configuration validates arguments beyond shape. — [elixir/lib/symphony_elixir/linear/agent_tool.ex:56-67](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/linear/agent_tool.ex#L56-L67) (verified)
  - *To reach the next level:* No validation layer applies to tool arguments.
- **D L0:** Shell, file writes, network and the read/write tracker tool are all enabled in the shipped workflow, and the tracker tool is always advertised. — [elixir/WORKFLOW.md:36-39](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L36-L39); [elixir/lib/symphony_elixir/tracker.ex:48-58](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/tracker.ex#L48-L58) (verified)
  - *To reach the next level:* Tool groups cannot be narrowed per task; write, exec and network are all on.
- **B L1:** Writes from shell commands are limited to the workspace by the sandbox, but reads cover the disk, network reaches any host, and the tracker tool reaches the whole workspace. — [elixir/lib/symphony_elixir/config/schema.ex:570-579](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/config/schema.ex#L570-L579); [elixir/WORKFLOW.md:36-39](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L36-L39) (verified)
  - *To reach the next level:* Tool reach is not bounded to the current project, ticket or quantity.
- **Cap:** none

### C4 Code-execution isolation — 0.33 (medium)

Codex runs each agent inside its own OS sandbox, configured by Symphony to 'workspace-write' so commands can only write inside the per-issue workspace; the sandbox itself is implemented in Codex, outside this repository. The shipped workflow turns network access on inside that sandbox, passes the operator's full environment to commands, and the default policy gives read access to the entire filesystem, so home-directory and environment credentials are reachable from inside. Workspace hooks (clone, dependency fetch, cleanup tasks) run directly on the host with Symphony's full environment and no sandbox. The sandbox mode and policy are plain workflow settings passed through without warning.

- **S L2:** Symphony requests Codex's workspace-write OS sandbox for every thread and turn; the sandbox implementation lives in Codex and was not verifiable in this repository. — [elixir/lib/symphony_elixir/config/schema.ex:193](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/config/schema.ex#L193); [elixir/lib/symphony_elixir/codex/app_server.ex:320-328](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/codex/app_server.ex#L320-L328) (inferred)
  - *To reach the next level:* Network is enabled inside the sandbox in the shipped workflow, and the boundary's implementation cannot be verified here.
- **C L1:** Codex commands go through the sandbox, but workspace hooks run on the host via a login shell in the workspace directory. — [elixir/lib/symphony_elixir/workspace.ex:397-405](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/workspace.ex#L397-L405); [elixir/WORKFLOW.md:22-29](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L22-L29) (verified)
  - *To reach the next level:* Workspace hooks and their child processes run on the host outside the sandbox.
- **D L2:** The sandbox is on by default, but any sandbox mode (including full access) or an explicit policy map from the workflow file is passed through unchanged without warning. — [elixir/lib/symphony_elixir/config/schema.ex:320-331](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/config/schema.ex#L320-L331) (verified)
  - *To reach the next level:* Disabling or widening the sandbox needs no loudly named flag and produces no warning.
- **B L0:** Inside the sandbox the agent can read the whole disk including home-directory credentials, has the operator's full environment, and has unrestricted network in the shipped workflow. — [elixir/lib/symphony_elixir/config/schema.ex:570-579](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/config/schema.ex#L570-L579); [elixir/WORKFLOW.md:34](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L34); [elixir/WORKFLOW.md:36-39](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L36-L39) (verified)
  - *To reach the next level:* Home directory and environment credentials are reachable from inside with network egress on.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Ticket titles and descriptions are rendered straight into the agent's prompt, and the workflow tells the agent to treat every PR review comment, human or bot, as blocking work, alongside repository files, command output and anything it fetches over the network. Nothing marks or limits these sources. In the same unattended session the agent can read the operator's files and credentials, send data anywhere over the network, push code, comment, and change any ticket in the workspace. A successful prompt injection therefore leads to data leaks and irreversible actions with no human involved.

- **S L0:** Nothing limits a hijacked agent; untrusted ticket text is rendered directly into the prompt and no code separates or flags it. — [elixir/lib/symphony_elixir/prompt_builder.ex:17-24](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/prompt_builder.ex#L17-L24); searched `rg -n -i 'untrusted|prompt.injection|provenance|taint'` in `elixir/lib` → 0 hits (No code distinguishes untrusted content.) (verified)
  - *To reach the next level:* No structural limit applies once untrusted content is read.
- **C L0:** Ticket text, PR comments from any reviewer or bot, repository content and tool results all enter context with the same standing as the operator's workflow prompt. — [elixir/WORKFLOW.md:180](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L180) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished.
- **D L0:** No mitigation exists to be enabled. — [elixir/WORKFLOW.md:35](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L35) (verified)
  - *To reach the next level:* No control exists.
- **B L0:** A hijacked agent can exfiltrate operator files and credentials over open network and take irreversible actions (pushes, comments, tracker mutations) unattended. — [elixir/WORKFLOW.md:36-39](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L36-L39); [elixir/lib/symphony_elixir/linear/agent_tool.ex:8-11](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/linear/agent_tool.ex#L8-L11); [elixir/lib/symphony_elixir/config/schema.ex:570-579](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/config/schema.ex#L570-L579) (verified)
  - *To reach the next level:* Untrusted content, sensitive data and egress are combined in one unattended session.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.05 (high)

The agent's working memory is a single 'Codex Workpad' comment on each ticket that it writes freely and every later run, retry or rework reads back as its plan; other people and agents can see and edit those comments too. Per-issue workspaces persist between runs. The shipped setup hook automatically marks the cloned repository's mise configuration as trusted, so repository-controlled tool configuration is honored by later host-side commands without an operator decision, and Codex loads the repository's own instruction and skill files. None of these paths is validated or versioned by Symphony.

- **S L0:** The model writes the persistent workpad with no validation and it is re-read as the plan; the shipped hook auto-trusts repository tool configuration. — [elixir/WORKFLOW.md:141-146](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L141-L146); [elixir/WORKFLOW.md:23-26](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L23-L26) (verified)
  - *To reach the next level:* No gating or provenance on persistent context, and no trust decision for repository-controlled configuration.
- **C L0:** No memory or configuration path is controlled. — [elixir/WORKFLOW.md:23-26](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L23-L26) (verified)
  - *To reach the next level:* No path is controlled.
- **D L1:** Separation between tickets' workpads is only a prompt convention; the tracker key can read and write every issue. — [elixir/WORKFLOW.md:281](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L281); [elixir/lib/symphony_elixir/linear/agent_tool.ex:8-11](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/linear/agent_tool.ex#L8-L11) (verified)
  - *To reach the next level:* Per-issue separation is not enforced in code.
- **B L0:** Poisoned workpad comments persist across runs, are visible to and editable by other users and agents, and drive tool use. — [elixir/WORKFLOW.md:141-146](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L141-L146); [elixir/WORKFLOW.md:136](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L136) (verified)
  - *To reach the next level:* Poisoned context persists across sessions and users and can trigger tool use.
- **Cap:** C6-REPOCONFIG — The shipped workflow hook marks the workspace repository's mise configuration as trusted automatically, so repository files configure host-side commands without an operator trust decision.

### C7 Third-party extensions — 0.00 (high)

Symphony itself loads no plugins, but the agent it runs can install and execute any package from the internet: the shipped workflow gives the sandbox network access and approves everything automatically, and the setup hook fetches dependencies on the host. When Codex asks whether to allow an MCP tool call, Symphony answers 'approve' on its own under the shipped policy. Nothing pins, verifies or confines what gets installed; it runs as the operator's user with the full environment.

- **S L0:** With network on and approvals automatic, model-chosen package installs run unverified; MCP tool-call approval prompts are answered automatically. — [elixir/WORKFLOW.md:36-39](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L36-L39); [elixir/lib/symphony_elixir/codex/app_server.ex:857-866](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/codex/app_server.ex#L857-L866) (verified)
  - *To reach the next level:* No pinning or integrity checks on what the agent installs or launches.
- **C L0:** No extension or package type is verified. — [elixir/WORKFLOW.md:23-26](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L23-L26); searched `rg -n -i 'sha256|checksum|integrity|signature'` in `elixir/lib` → 3 hits (Hits are the workspace-name hash and static dashboard asset digests, not extension verification.) (verified)
  - *To reach the next level:* No extension type is verified.
- **D L0:** Installs need no consent, and MCP approval prompts are auto-answered under the shipped policy. — [elixir/lib/symphony_elixir/codex/app_server.ex:796-822](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/codex/app_server.ex#L796-L822); [elixir/WORKFLOW.md:35](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L35) (verified)
  - *To reach the next level:* No explicit consent step for installs or MCP tools.
- **B L0:** Installed code runs as the operator with the full inherited environment and network. — [elixir/WORKFLOW.md:34](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L34); [elixir/WORKFLOW.md:36-39](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L36-L39) (verified)
  - *To reach the next level:* No per-extension confinement or environment scrubbing.
- **Cap:** C7-RCELOAD — In the shipped workflow the agent can install and run arbitrary remote packages over open network with no consent step.

### C8 Secrets & sensitive-data protection — 0.20 (high)

Tracker keys come from environment variables and are kept host-side: Symphony removes them from the Codex process environment and executes tracker calls itself, so the key never enters the prompt. That is the only protection. Every other credential in the operator's environment is passed to Codex commands under the shipped workflow, hooks receive everything including the tracker key, nothing is masked or filtered, and the disk log handler records all log levels, including raw Codex stream lines at debug level. There is no telemetry.

- **S L1:** Tracker keys are read from environment variables and unset before Codex starts; nothing is masked or filtered elsewhere. — [elixir/lib/symphony_elixir/codex/app_server.ex:240-251](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/codex/app_server.ex#L240-L251); [elixir/lib/symphony_elixir/codex/app_server.ex:207](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/codex/app_server.ex#L207) (verified)
  - *To reach the next level:* No secret masking or filtering on logs or tool output.
- **C L1:** Only the Codex child environment is protected; hooks inherit Symphony's full environment and logs are not filtered. — [elixir/lib/symphony_elixir/workspace.ex:397-405](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/workspace.ex#L397-L405); [elixir/lib/symphony_elixir/codex/app_server.ex:930](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/codex/app_server.ex#L930) (verified)
  - *To reach the next level:* Logs, hook environments and other subprocess paths are unprotected.
- **D L1:** No telemetry, but the disk log handler accepts all levels and debug lines include raw stream content with no secret filtering. — [elixir/lib/symphony_elixir/log_file.ex:68-71](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/log_file.ex#L68-L71); searched `rg -n -i 'sentry|posthog|telemetry'` in `elixir/lib elixir/mix.exs` → 1 hits (Single hit is Phoenix's local Plug.Telemetry instrumentation, not an exporter.) (verified)
  - *To reach the next level:* Verbose logging is on by default with no secret filtering.
- **B L0:** Long-lived personal tracker keys and the operator's environment credentials are reachable by commands and hooks. — [elixir/WORKFLOW.md:34](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L34); [elixir/lib/symphony_elixir/config/schema.ex:411-412](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/config/schema.ex#L411-L412) (verified)
  - *To reach the next level:* Keys are long-lived, broadly scoped and reachable by subprocesses.
- **Cap:** none

### C9 Audit & traceability — 0.30 (high)

Symphony writes a rotating log file outside the workspace with lifecycle events: dispatch, session start and end, retries, hook runs and failures, tagged with issue and session identifiers. Individual commands, file changes and tracker tool calls (including mutations Symphony executes on the agent's behalf) are kept only as the latest event in memory for the dashboard and are not written to its log. Codex may keep its own session transcript, but that is outside this repository. The log rotates after 50 MB and nothing is tamper-evident.

- **S L1:** Unstructured lifecycle log lines; tool calls are not recorded by Symphony. — [elixir/lib/symphony_elixir/codex/app_server.ex:97](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/codex/app_server.ex#L97); searched `rg -n 'Logger\.(info|warning|error|debug)'` in `elixir/lib/symphony_elixir/linear/agent_tool.ex elixir/lib/symphony_elixir/codex/dynamic_tool.ex elixir/lib/symphony_elixir/tracker.ex` → 0 hits (Tracker tool execution path writes no log.) (verified)
  - *To reach the next level:* No structured per-tool-call record with arguments and results.
- **C L1:** Only the run lifecycle and hooks are logged; tool and command activity is kept in memory only. — [elixir/lib/symphony_elixir/orchestrator.ex:167-185](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/orchestrator.ex#L167-L185) (verified)
  - *To reach the next level:* Tool calls, including tracker mutations, are not recorded.
- **D L2:** Logging is on by default to ./log under the launch directory, outside the sandbox's writable workspace, but host-side hooks run as the same user. — [elixir/lib/symphony_elixir/log_file.ex:9](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/log_file.ex#L9) (verified)
  - *To reach the next level:* The log is writable by the same user that runs hooks.
- **B L1:** OTP disk_log wrap handler with five 10 MB files; older records are overwritten and write failures are not surfaced to runs. — [elixir/lib/symphony_elixir/log_file.ex:68-78](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/log_file.ex#L68-L78); [elixir/lib/symphony_elixir/log_file.ex:10-11](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/log_file.ex#L10-L11) (verified)
  - *To reach the next level:* Records are not flushed durably per action or replayable.
- **Cap:** none

### C10 Limits & kill switch — 0.33 (high)

Each agent invocation runs at most 20 Codex turns, a turn is abandoned after an hour without output, stalled runs are restarted after five minutes, hooks time out after 60 seconds, and at most 10 agents run at once. But when an invocation ends with the ticket still active, Symphony schedules a continuation a second later, so there is no cap on total turns, time or spend per ticket, and no token or cost budget at all. The agent can keep itself running by leaving its ticket active or by filing new tickets that start new runs. Moving a ticket to a terminal state stops its agent and closes the Codex process pipe.

- **S L1:** Per-invocation turn cap with idle timeouts; continuation re-dispatches indefinitely and there is no token or cost cap. — [elixir/lib/symphony_elixir/agent_runner.ex:113-130](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/agent_runner.ex#L113-L130); [elixir/lib/symphony_elixir/orchestrator.ex:212-222](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/orchestrator.ex#L212-L222) (verified)
  - *To reach the next level:* No total turn, wall-clock or cost cap per ticket.
- **C L2:** The top-level loop plus timeouts on turns, protocol reads and hooks. — [elixir/lib/symphony_elixir/config/schema.ex:195-197](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/config/schema.ex#L195-L197); [elixir/lib/symphony_elixir/workspace.ex:407-416](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/workspace.ex#L407-L416) (verified)
  - *To reach the next level:* Runs spawned from tickets the agent files do not share any budget.
- **D L1:** Defaults are moderate per invocation, but the model can extend its run by keeping the ticket active or by creating new tickets with its tracker tool. — [elixir/WORKFLOW.md:90](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/WORKFLOW.md#L90); [elixir/lib/symphony_elixir/config/schema.ex:152-153](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/config/schema.ex#L152-L153) (verified)
  - *To reach the next level:* The model can extend or reset its limits through the tracker.
- **B L1:** No spend ceiling; stopping terminates the task and closes the port without killing the process group. — [elixir/lib/symphony_elixir/orchestrator.ex:723-731](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/orchestrator.ex#L723-L731); [elixir/lib/symphony_elixir/codex/app_server.ex:966-980](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/codex/app_server.ex#L966-L980) (verified)
  - *To reach the next level:* No ceiling on total spend or time, and stop does not kill spawned process groups.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Ticket text rendered into the prompt (elixir/lib/symphony_elixir/prompt_builder.ex:17-24) and PR comments from any reviewer or bot (elixir/WORKFLOW.md:180) · [B] sensitive data/systems: Full-disk read access in the sandbox policy (elixir/lib/symphony_elixir/config/schema.ex:574), full inherited environment (elixir/WORKFLOW.md:34), operator's Linear key behind linear_graphql (elixir/lib/symphony_elixir/linear/agent_tool.ex:8-11) · [C] state change / egress: Network on in the sandbox (elixir/WORKFLOW.md:39), raw GraphQL mutations (elixir/lib/symphony_elixir/linear/agent_tool.ex:56-67), auto-approved requests (elixir/lib/symphony_elixir/codex/app_server.ex:55) · Same default session? Yes

## Highest-impact improvements
1. Ship WORKFLOW.md with network access off, without inheriting the full environment, and with sandbox read access limited to the workspace. — C4 B L0→L2, +0.100 before caps (Playbook 3 step 1)
2. Route Codex approval requests to a human through the dashboard, showing the exact command or diff, instead of auto-accepting or failing them. — C2 S L0→L3, +0.225 before caps (Playbook 5)
3. Replace raw linear_graphql with narrow tools (comment on, update state of, the current issue) or an operation allowlist bound to the current issue. — C3 S L0→L3, +0.225 before caps (Playbook 3)
4. Run workspace hooks with a scrubbed environment and inside the same sandbox as the agent. — C4 C L1→L2, +0.075 before caps (Playbook 3 step 1)
5. Add a per-ticket total turn and token budget that continuations and agent-filed tickets count against. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored configuration is the shipped elixir/WORKFLOW.md that the README tells users to copy and run. Without it, the built-in schema defaults reject escalation requests instead of auto-accepting them and turn network off in the sandbox, which would raise C4 and C5 blast radius; there is still no human approval path.
- The Codex sandbox, Codex session transcripts and Codex's loading of AGENTS.md, skills and MCP servers are implemented in the separately scored Codex CLI, not in this repository; statements about them are inferred.
- Only the Linear adapter was scored. The GitHub, GitLab, Jira and Asana adapters expose similar raw-API tools; with GitHub Issues on a public repository, issues from anyone could trigger runs unless required labels or assignee filters are set.
- SPEC.md is a language-agnostic specification; only the Elixir reference implementation was scored.
- No text aimed at AI reviewers was found in the repository.
