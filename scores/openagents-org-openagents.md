# Defense-in-Depth Score: OpenAgents

**Repo:** https://github.com/openagents-org/openagents · **Commit:** `812890186609158b5caf61263723aaeee67c6f3a` · **Reviewed:** 2026-10-04
**What it is:** Launcher, daemon and shared workspace that connect local coding agents (Claude Code, Codex, OpenClaw and others) into a collaborative multi-agent workspace.
**Category:** Agent Frameworks
**Scored configuration:** The `agn` launcher/daemon (packages/agent-connector) with a Claude Code agent in its default skills tool mode and execute mode, connected to a workspace via token; other adapters checked for the same pattern.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 1.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L0 | L1 | L0 | L0 | 0.07 | — | **0.07** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L0 | 0.05 | C6-REPOCONFIG | **0.05** | Medium |
| C7 | Third-party extensions | L1 | L0 | L0 | L0 | 0.07 | — | **0.07** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | C8-MODELSECRETS | **0.20** | High |
| C9 | Audit & traceability | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |


OpenAgents turns local coding agents into workspace participants, and to do so it switches off every agent's own approval prompts and sandbox while adding no gate of its own. Any message that reaches the agent through the workspace, from any link holder, another agent, or content read in the shared browser, can run shell commands on the user's machine with the user's full environment. The owner-level workspace token is also written into model-read skill files, and remote control events can install unpinned skills. Run it only on a disposable machine or account.

## Critical gaps
- Agents run with the user's full environment and an owner-equivalent workspace token, with no narrowing. (ASI03, T3; C1) — [packages/agent-connector/src/daemon.js:1562](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/daemon.js#L1562); [workspace/backend/app/access.py:42-43](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/workspace/backend/app/access.py#L42-L43)
- Commands run unsandboxed on the host; the Codex adapter explicitly disables Codex's own sandbox. (ASI05, T11; C4) — [packages/agent-connector/src/adapters/codex.js:315](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/codex.js#L315); [packages/agent-connector/src/daemon.js:1562](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/daemon.js#L1562)
- Untrusted workspace content (peer agents, shared browser, files, any link holder) can drive agents that run shell with permissions skipped, enabling unattended exfiltration and irreversible actions. (ASI01, LLM01, T6; C5) — [packages/agent-connector/src/adapters/claude.js:598](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L598); [packages/agent-connector/src/adapters/base.js:1094](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/base.js#L1094)
- Workspace control events install skills (with scripts) into the agent's auto-loaded skills directory without a local trust decision. (ASI06, T1, ASI04; C6) — [packages/agent-connector/src/adapters/base.js:754](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/base.js#L754); [packages/agent-connector/src/skill-installer.js:140-142](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/skill-installer.js#L140-L142)
- Unpinned, remotely-triggered skills run inside a permission-skipping agent with the user's full environment. (ASI04, T17; C7) — [packages/agent-connector/src/skill-installer.js:183](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/skill-installer.js#L183); [packages/agent-connector/src/daemon.js:1562](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/daemon.js#L1562)
- The owner-equivalent workspace token is written into model-read skill files and prompts by default. (LLM02, ASI03; C8) — [packages/agent-connector/src/adapters/workspace-prompt.js:274](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/workspace-prompt.js#L274); [packages/agent-connector/src/adapters/claude.js:537](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L537); [packages/agent-connector/src/adapters/claude.js:95](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L95)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

The daemon launches each coding CLI with the operator's entire shell environment plus stored provider keys, so cloud, GitHub and other ambient credentials reach a model running with all permissions skipped. The workspace connection uses a single workspace token that the backend treats as owner-equivalent and that is shared by agents, daemons and share links. There is no per-tool or per-request authorization and no narrowing of the user's authority. A hijacked agent holds everything the user holds on that machine plus owner rights on the workspace.

- **S L0:** Agent CLIs run with the operator's full ambient authority and an owner-equivalent workspace token. — [packages/agent-connector/src/daemon.js:1562](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/daemon.js#L1562); [workspace/backend/app/access.py:42-43](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/workspace/backend/app/access.py#L42-L43) (verified)
  - *To reach the next level:* No dedicated or role-scoped identity for agents; read and write share one owner-level token.
- **C L0:** No authorization layer sits between the model and its tools; every subprocess inherits the full process environment. — [packages/agent-connector/src/daemon.js:1562](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/daemon.js#L1562); [packages/agent-connector/src/adapters/claude.js:598](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L598) (verified)
  - *To reach the next level:* No authorization check on any tool path; subprocesses are not given a scrubbed environment.
- **D L0:** Default install runs agents as the OS user with the full environment and owner-level workspace token. — [packages/agent-connector/src/daemon.js:1562](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/daemon.js#L1562); [workspace/backend/app/access.py:42-43](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/workspace/backend/app/access.py#L42-L43) (verified)
  - *To reach the next level:* No narrower default identity exists.
- **B L0:** Compromise yields the user's local account credentials across services and owner control of the workspace. — [packages/agent-connector/src/daemon.js:1562](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/daemon.js#L1562); [workspace/backend/app/access.py:42-43](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/workspace/backend/app/access.py#L42-L43) (verified)
  - *To reach the next level:* Credentials are not scoped to one system or tenant.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

Every shipped CLI adapter disables its runtime's own approval prompts (Claude Code with --dangerously-skip-permissions, Codex with --dangerously-bypass-approvals-and-sandbox, others with --yolo or auto-approve), and comments state the daemon is meant to be the approval boundary. The daemon contains no approval step: workspace messages are dispatched straight to the CLI. Shell, file writes, public tunnels and scheduled routines all run without a human seeing the exact call. A plan mode exists but is a mode toggle, not a per-call gate.

- **S L0:** No approval mechanism; the agents' own approval prompts are switched off. — [packages/agent-connector/src/adapters/claude.js:598](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L598); [packages/agent-connector/src/adapters/antigravity-stream.js:22-24](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/antigravity-stream.js#L22-L24); searched `rg -n -i 'approv|confirm'` in `packages/agent-connector/src/adapters/base.js packages/agent-connector/src/adapters/claude.js` → 6 hits (all hits are comments about confirming process death or knowledge-entry existence; none is an action approval gate) (verified)
  - *To reach the next level:* No per-call human approval of any kind.
- **C L0:** The most powerful path (Bash with skipped permissions) is ungated, as is every other tool. — [packages/agent-connector/src/adapters/claude.js:511-512](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L511-L512); [packages/agent-connector/src/adapters/codex.js:315](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/codex.js#L315) (verified)
  - *To reach the next level:* Shell and write tools are not routed through any gate.
- **D L0:** Execute mode with approvals disabled is the default. — [packages/agent-connector/src/adapters/base.js:140](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/base.js#L140); [packages/agent-connector/src/adapters/claude.js:598](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L598) (verified)
  - *To reach the next level:* Approval is not available, let alone on by default.
- **B L0:** Ungated actions include arbitrary shell on the user's machine, public tunnels, and outbound requests, with no undo. — [packages/agent-connector/src/adapters/claude.js:511-512](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L511-L512); [packages/agent-connector/src/mcp-server.js:263](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/mcp-server.js#L263) (verified)
  - *To reach the next level:* No checkpoints, previews or bounds on consequential actions.
- **Cap:** none

### C3 Tool & action scoping — 0.07 (high)

The default tool set given to Claude Code is Read, Write, Edit, Bash, Glob and Grep plus workspace tools, with permissions skipped, so arbitrary shell and network access are part of designed use. The connector's own comment notes the allowlist is a suggestion, not a boundary, in execute mode. The workspace fetch and file-from-URL endpoints are well protected against SSRF on the server, but that does not constrain the local shell. The result is an agent whose tools reach the whole machine.

- **S L0:** Raw shell passthrough is in the default tool set and the allowlist is unenforced under skipped permissions. — [packages/agent-connector/src/adapters/claude.js:511-512](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L511-L512); [packages/agent-connector/src/cli.js:1203-1204](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/cli.js#L1203-L1204) (verified)
  - *To reach the next level:* Shell is not replaced or bounded; no argument validation for local tools.
- **C L1:** Only the server-side workspace fetch path validates URLs (SSRF-safe, redirect-rechecked). — [workspace/backend/app/net_security.py:3](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/workspace/backend/app/net_security.py#L3); [packages/agent-connector/src/mcp-server.js:143](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/mcp-server.js#L143) (verified)
  - *To reach the next level:* Built-in local tools (Bash, Write) carry no validation.
- **D L0:** Write, exec and network tools are all enabled by default. — [packages/agent-connector/src/adapters/claude.js:511-512](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L511-L512); [packages/agent-connector/src/adapters/base.js:140](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/base.js#L140) (verified)
  - *To reach the next level:* No read-only default tool set.
- **B L0:** General-purpose shell against the whole machine and any host. — [packages/agent-connector/src/adapters/claude.js:511-512](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L511-L512) (verified)
  - *To reach the next level:* Tool reach is not confined to the working directory.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

Model-chosen commands run as plain host subprocesses of the daemon, in the agent's working directory, with the full user environment. The connector adds no container or OS sandbox, and for Codex it explicitly passes the flag that removes Codex's own sandbox. One adapter (DeepSeek) deliberately keeps its runtime's workspace-write sandbox, but that is the exception. An injected command has the same reach as the user.

- **S L0:** No isolation primitive; commands run as the user on the host. — searched `rg -n -i 'sandbox|docker|bwrap|landlock|seatbelt'` in `packages/agent-connector/src/adapters/claude.js packages/agent-connector/src/adapters/base.js packages/agent-connector/src/daemon.js` → 0 hits; [packages/agent-connector/src/adapters/codex.js:315](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/codex.js#L315) (verified)
  - *To reach the next level:* No container, OS sandbox or separate low-privilege user.
- **C L0:** No execution path is sandboxed. — searched `rg -n -i 'sandbox|docker|bwrap|landlock|seatbelt'` in `packages/agent-connector/src/adapters/claude.js packages/agent-connector/src/adapters/base.js packages/agent-connector/src/daemon.js` → 0 hits (verified)
  - *To reach the next level:* The main exec tool is not sandboxed.
- **D L0:** No sandbox in the default configuration; Codex's is explicitly disabled. — [packages/agent-connector/src/adapters/codex.js:315](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/codex.js#L315) (verified)
  - *To reach the next level:* Isolation is not available by default.
- **B L0:** Host-equivalent: the process has the home directory, all files and the full credential-bearing environment. — [packages/agent-connector/src/daemon.js:1562](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/daemon.js#L1562); [packages/agent-connector/src/adapters/claude.js:598](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L598) (verified)
  - *To reach the next level:* Workspace-only mount and a scrubbed environment are absent.
- **Cap:** none

### C5 Untrusted input blast radius — 0.25 (high)

Agents act on any message delivered to them in the workspace, from any person holding the workspace link or token and from other agents, with no check that the sender is the agent's owner. Untrusted content arrives through the shared browser, web fetches, shared files and peer-agent messages, and the agent can then run shell commands, write files, open public tunnels and make outbound requests with no human step. The only structural defense found is a data fence around pinned knowledge-base entries, which is a prompt-level marker. A successful injection can leak local secrets and take irreversible actions unattended.

- **S L1:** Only spotlighting: pinned knowledge entries are wrapped in a 'this is DATA' fence. — [packages/agent-connector/src/adapters/workspace-prompt.js:802](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/workspace-prompt.js#L802) (verified)
  - *To reach the next level:* No capability is disabled or gated after untrusted content is read.
- **C L1:** The fence covers knowledge entries only; chat messages, peer agents, fetched pages and files enter undistinguished. — [packages/agent-connector/src/adapters/workspace-prompt.js:802](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/workspace-prompt.js#L802); [packages/agent-connector/src/adapters/base.js:1094](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/base.js#L1094) (verified)
  - *To reach the next level:* Tool results, files and other agents' messages are not distinguished.
- **D L2:** The fence is always applied and nothing read can switch it off. — [packages/agent-connector/src/adapters/workspace-prompt.js:802](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/workspace-prompt.js#L802) (verified)
  - *To reach the next level:* The only control is prompt text, so the ceiling is one level above its strength.
- **B L0:** A hijacked agent can exfiltrate via shell/curl or tunnels and take irreversible host actions with no human involved. — [packages/agent-connector/src/adapters/claude.js:598](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L598); [packages/agent-connector/src/mcp-server.js:263](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/mcp-server.js#L263) (verified)
  - *To reach the next level:* Neither exfiltration nor irreversible actions require approval.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.05 (medium)

Several persistence paths can carry an injection forward. Any workspace token holder can send a skill.install control event, and the daemon then downloads a skill (including bundled scripts) from GitHub or from an uploaded workspace file into the agent's skills directory, where the CLI auto-loads it, with no local trust decision. Agents can write shared knowledge entries that are pinned into later prompts as 'settled decisions', and can create recurring routines that re-trigger themselves. Knowledge is scoped per workspace but shared by all of its agents and users.

- **S L0:** Workspace-sent control events install skills into the agent's auto-loaded skills directory with no prompt; the model can write knowledge that is re-injected as pinned context. — [packages/agent-connector/src/adapters/base.js:754](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/base.js#L754); [packages/agent-connector/src/mcp-server.js:465](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/mcp-server.js#L465) (verified)
  - *To reach the next level:* No gate, validation or trust decision on skill installs or memory writes.
- **C L0:** No memory, skill or routine path is controlled. — [packages/agent-connector/src/adapters/base.js:754](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/base.js#L754); [packages/agent-connector/src/mcp-server.js:373](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/mcp-server.js#L373) (verified)
  - *To reach the next level:* Not even the main store (knowledge) has gated writes.
- **D L1:** Knowledge and routines are namespaced per workspace, but shared by every agent and user in it. — [packages/agent-connector/src/mcp-server.js:465](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/mcp-server.js#L465); [packages/agent-connector/src/mcp-server.js:373](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/mcp-server.js#L373) (inferred)
  - *To reach the next level:* No per-user or per-agent namespace; isolation is only at workspace level.
- **B L0:** Poisoned skills, knowledge and routines persist across sessions and users and can trigger tool use. — [packages/agent-connector/src/mcp-server.js:373](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/mcp-server.js#L373); [packages/agent-connector/src/adapters/base.js:754](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/base.js#L754) (verified)
  - *To reach the next level:* Persistence is not session-scoped or review-gated.
- **Cap:** C6-REPOCONFIG — Files and control events from the shared workspace install skills (instructions plus scripts) into the agent's auto-loaded skills directory without an explicit local trust decision.

### C7 Third-party extensions — 0.07 (high)

Skills are fetched from a GitHub repository's main or master branch (or from an uploaded workspace file) with no version pin, hash or signature. Installation is triggered remotely by a workspace control event, not by a local consent step. Installed skills run inside the coding CLI as the same user with the full environment and skipped permissions. Agent runtimes themselves come from the vendor's own registry endpoint.

- **S L1:** Skill sources are chosen in the workspace but unpinned (main/master at install time). — [packages/agent-connector/src/skill-installer.js:183](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/skill-installer.js#L183); searched `rg -n 'sha256|integrity|checksum'` in `packages/agent-connector/src/skill-installer.js` → 0 hits (verified)
  - *To reach the next level:* No version pin or integrity check.
- **C L0:** No extension type is verified. — searched `rg -n 'sha256|integrity|checksum'` in `packages/agent-connector/src/skill-installer.js` → 0 hits (verified)
  - *To reach the next level:* Skill installs and uploaded packages carry no verification.
- **D L0:** A remote workspace control event installs skills automatically, with no local consent. — [packages/agent-connector/src/adapters/base.js:754](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/base.js#L754) (verified)
  - *To reach the next level:* Install does not require explicit local approval showing what will run.
- **B L0:** Skills run inside the agent CLI as the user with the full environment and all credentials. — [packages/agent-connector/src/daemon.js:1562](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/daemon.js#L1562); [packages/agent-connector/src/adapters/claude.js:598](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L598) (verified)
  - *To reach the next level:* No separate process or scrubbed environment for extensions.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.20 (high)

In the default skills mode the owner-equivalent workspace token is written in plain text into a SKILL.md file inside the agent's working directory and becomes part of what the model reads. Provider API keys are stored as plain text env files with no permission tightening, and the full environment is passed to the CLI. Error and stderr diagnostics are redacted, but tool-call previews (commands, paths, the first 100 characters of written content) are posted unredacted to the workspace. No telemetry SDK was found in the connector.

- **S L1:** Secrets from env files and env vars; redaction only on error diagnostics; token placed into model-read files. — [packages/agent-connector/src/adapters/workspace-prompt.js:274](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/workspace-prompt.js#L274); [packages/agent-connector/src/env.js:51](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/env.js#L51); searched `rg -n chmod` in `packages/agent-connector/src/env.js packages/agent-connector/src/config.js` → 0 hits (verified)
  - *To reach the next level:* No type-level masking, keychain, or restrictive file permissions on stored keys.
- **C L1:** Only stderr/error diagnostics are redacted. — [packages/agent-connector/src/adapters/claude.js:1036](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L1036); [packages/agent-connector/src/adapters/claude.js:895](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L895) (verified)
  - *To reach the next level:* Model-bound content, tool previews and subprocess env are unprotected.
- **D L1:** No telemetry, but tool-input previews stream to the workspace by default and cannot be turned off. — [packages/agent-connector/src/adapters/claude.js:895](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L895) (verified)
  - *To reach the next level:* Tool previews are not redacted or minimised by default.
- **B L0:** Long-lived owner-level workspace token and provider keys are reachable by the model and every subprocess. — [packages/agent-connector/src/adapters/workspace-prompt.js:274](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/workspace-prompt.js#L274); [packages/agent-connector/src/daemon.js:1562](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/daemon.js#L1562) (verified)
  - *To reach the next level:* Keys are neither scoped nor short-lived.
- **Cap:** C8-MODELSECRETS — The default skills mode writes the workspace token into a SKILL.md the model reads, and the same API prompt template embeds it as an auth header.

### C9 Audit & traceability — 0.20 (high)

For Claude, each tool call is posted to the workspace as a status line with a truncated input preview, and the daemon writes an unstructured log to ~/.openagents/daemon.log. There is no structured record of arguments and results, no attribution of which human or agent requested the action, and failures to post are silently swallowed. The local log is writable by the agent's own shell, and the log rotates at 10 MB keeping one backup.

- **S L1:** Unstructured status lines with truncated tool previews and a plain-text daemon log. — [packages/agent-connector/src/adapters/claude.js:895](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L895); [packages/agent-connector/src/daemon.js:1780](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/daemon.js#L1780) (verified)
  - *To reach the next level:* No structured per-call record with arguments, results and timestamps.
- **C L1:** Main CLI tool path only; workspace-triggered control actions and skill installs are only in the local log. — [packages/agent-connector/src/adapters/claude.js:895](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L895) (verified)
  - *To reach the next level:* Not all tool calls and control actions are recorded in one record.
- **D L1:** On by default, but the local log sits where the agent's unsandboxed shell can edit it. — [packages/agent-connector/src/daemon.js:1780](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/daemon.js#L1780); [packages/agent-connector/src/adapters/claude.js:598](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L598) (verified)
  - *To reach the next level:* Record is not written by a component the model cannot control.
- **B L0:** Status-post failures are swallowed and actions proceed. — [packages/agent-connector/src/daemon.js:1779-1782](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/daemon.js#L1779-L1782) (verified)
  - *To reach the next level:* Errors are not surfaced and records are not durable per action.
- **Cap:** none

### C10 Limits & kill switch — 0.25 (high)

The workspace Stop control kills the CLI's whole process group, which is a real halt. Beyond that there is no step, cost or wall-clock cap from OpenAgents: only a five-minute silence watchdog and a one-hour idle release. Agents can create recurring routines and timers that re-trigger themselves after a run ends, and those continue until cancelled.

- **S L1:** Process-group kill on stop and a silence watchdog; no step, cost or wall-clock budget. — [packages/agent-connector/src/adapters/claude.js:240](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L240); [packages/agent-connector/src/adapters/claude.js:103-105](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L103-L105); searched `rg -n -i 'max_?turns|max_?steps|max_?iterations|max_?cost|budget'` in `packages/agent-connector/src/adapters/base.js packages/agent-connector/src/adapters/claude.js` → 2 hits (CLAUDE_CODE_MAX_TURNS is only preserved in the env allowlist (not set); the other hit is an HTTP round-trip comment) (verified)
  - *To reach the next level:* No iteration cap plus wall-clock or cost cap enforced in code.
- **C L1:** Watchdog applies to the top-level CLI process only. — [packages/agent-connector/src/adapters/claude.js:103-105](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L103-L105); [packages/agent-connector/src/mcp-server.js:373](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/mcp-server.js#L373) (verified)
  - *To reach the next level:* Routines and timers created by the agent are not bounded.
- **D L1:** Defaults are silence-based only, and the model can schedule further runs of itself. — [packages/agent-connector/src/mcp-server.js:373](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/mcp-server.js#L373); searched `rg -n -i 'max_?turns|max_?steps|max_?iterations|max_?cost|budget'` in `packages/agent-connector/src/adapters/base.js packages/agent-connector/src/adapters/claude.js` → 2 hits (CLAUDE_CODE_MAX_TURNS is only preserved in the env allowlist (not set); the other hit is an HTTP round-trip comment) (verified)
  - *To reach the next level:* No sensible run budget by default.
- **B L1:** A continuously active run has no ceiling, and self-created routines keep firing after a stop. — [packages/agent-connector/src/mcp-server.js:373](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/mcp-server.js#L373); [packages/agent-connector/src/adapters/claude.js:240](https://github.com/openagents-org/openagents/blob/812890186609158b5caf61263723aaeee67c6f3a/packages/agent-connector/src/adapters/claude.js#L240) (verified)
  - *To reach the next level:* Stopping does not cancel scheduled routines; no tight per-run ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: workspace messages from any token holder or agent, dispatched without sender checks (packages/agent-connector/src/adapters/base.js:1094) · [B] sensitive data/systems: full process env and owner-level workspace token (packages/agent-connector/src/daemon.js:1562, workspace-prompt.js:274) · [C] state change / egress: Bash/Write with permissions skipped and public tunnels (packages/agent-connector/src/adapters/claude.js:598, mcp-server.js:263) · Same default session? Yes

## Highest-impact improvements
1. Pass a scrubbed environment to agent CLIs (only the provider key and the agent's own config), not the whole process.env. — C1 C L0→L1, +0.075 before caps (Playbook 4)
2. Keep the workspace token out of SKILL.md and prompts: use MCP mode with the token only in the MCP server's environment, or a shell variable reference. — C8 S L1→L2, +0.075 before caps (Playbook 4)
3. Require local operator approval, showing source and commit, before any workspace-requested skill install, and pin skills to a commit hash. — C7 D L0→L2, +0.100 before caps (Playbook 3)
4. Implement the daemon-side approval boundary the adapter comments promise: route shell and write tools through a per-call approval shown in the workspace to the agent's owner. — C2 S L0→L3, +0.225 before caps (Playbook 5)
5. Stop removing runtimes' own sandboxes (e.g. use Codex workspace-write instead of --dangerously-bypass-approvals-and-sandbox). — C4 S L0→L3, +0.225 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the agent-connector daemon with the Claude adapter as representative; ~25 other adapters were spot-checked by grep for approval/sandbox flags only. The Python SDK, Electron launcher UI and workspace frontend were not reviewed in depth.
- Workspace backend reviewed only for its access model (access.py) and SSRF module; whether agent-posted status messages are durably stored server-side was not verified.
- Behaviour of the third-party CLIs (e.g. Claude Code auto-loading skills from .claude/skills) is relied on from their documented behaviour, not from code in this repo.
- No text aimed at AI reviewers was found in the repository.
