# Defense-in-Depth Score: Maestro

**Repo:** https://github.com/runmaestro/maestro · **Commit:** `54ffa324752415d7015e0dda3230c6811d4f52c3` (0.17.6) · **Reviewed:** 2026-10-04
**What it is:** Cross-platform Electron desktop app that orchestrates fleets of coding-agent CLIs (Claude Code, Codex, OpenCode, Factory Droid, Copilot) with Auto Run playbooks, Cue event automations, group chat and mobile remote control.
**Category:** Coding
**Scored configuration:** Desktop app, fresh install with shipped defaults (Encore features at their declared defaults), agents spawned through Maestro's built-in agent definitions.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 1.2 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | C2-POWERBYPASS | **0.00** | High |
| C3 | Tool & action scoping | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L0 | 0.17 | — | **0.17** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C9 | Audit & traceability | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


Maestro launches every supported coding agent with its permission prompts and sandbox switched off, which its own code describes as a requirement, and gives each agent the user's full environment. Any prompt injection from a repository, web page or GitHub issue can therefore run commands, read credentials and send them anywhere with no human in the loop. Project-level .maestro/cue.yaml files can also add shell and agent automations without a trust prompt. Use it only inside a disposable VM or container holding no sensitive credentials.

## Critical gaps
- Every agent is spawned with its approval gate and sandbox disabled (--dangerously-skip-permissions, --dangerously-bypass-approvals-and-sandbox, --skip-permissions-unsafe, --allow-all). (ASI02, ASI09, T10; C2) — [src/main/agents/definitions.ts:169-175](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L169-L175); [src/main/agents/definitions.ts:251](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L251)
- Agent and Cue shell execution runs on the host as the user with no isolation, and Codex's own sandbox is bypassed. (ASI05, T11; C4) — [src/main/agents/definitions.ts:251](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L251); [src/main/cue/cue-shell-executor.ts:6](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/cue/cue-shell-executor.ts#L6)
- A successful injection can exfiltrate secrets and take irreversible actions unattended; the only filter is an optional, fail-open classifier. (ASI01, LLM01, T6; C5) — [src/main/cue/cue-susfactor.ts:25-28](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/cue/cue-susfactor.ts#L25-L28); [src/main/agents/definitions.ts:169-175](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L169-L175)
- Agents inherit the full process environment, giving a hijacked agent the user's credentials across services. (ASI03, T3; C1) — [src/main/process-manager/utils/envBuilder.ts:295](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/process-manager/utils/envBuilder.ts#L295)
- Workspace .maestro/cue.yaml is auto-discovered and can define shell-command and agent automations without a trust decision. (ASI06, T1; C6) — [src/main/cue/cue-yaml-loader.ts:133](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/cue/cue-yaml-loader.ts#L133); [src/main/cue/cue-shell-executor.ts:6](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/cue/cue-shell-executor.ts#L6)
- Marketplace playbooks (including script assets) are fetched unpinned from a main branch and executed by full-permission agents with the user's environment. (ASI04, T17; C7) — [src/main/services/marketplace-service.ts:32](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/services/marketplace-service.ts#L32); [src/main/process-manager/utils/envBuilder.ts:295](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/process-manager/utils/envBuilder.ts#L295)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

Maestro runs every agent as the desktop user and hands each one a copy of its entire environment, so the agents inherit every API key, cloud credential, gh login and SSH agent socket the user has. Nothing in Maestro maps actions to a narrower identity or checks authorization per request. A remote-control web server also starts at every launch, listens on all network interfaces over plain HTTP, and accepts any client that presents the URL token, which can then run terminal input and change settings. A hijacked agent or stolen link therefore carries the user's full account authority.

- **S L0:** Agents run with the operator's full ambient authority; the whole process environment is copied into each agent's environment. — [src/main/process-manager/utils/envBuilder.ts:295](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/process-manager/utils/envBuilder.ts#L295) (verified)
  - *To reach the next level:* No scoped or per-tool identity; credentials are not narrowed before agents receive them.
- **C L0:** No authorization layer exists on any agent path; the remote-control server authorises every command with one shared URL token. — [src/main/web-server/WebServer.ts:219](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/web-server/WebServer.ts#L219); [src/main/web-server/handlers/messageHandlers.ts:604](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/web-server/handlers/messageHandlers.ts#L604); [src/main/web-server/handlers/messageHandlers.ts:704](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/web-server/handlers/messageHandlers.ts#L704) (verified)
  - *To reach the next level:* No path checks authorization against a scoped identity or principal.
- **D L0:** Default install grants agents full user authority and starts a LAN-bound control server at boot. — [src/main/index.ts:846](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/index.ts#L846); [src/main/web-server/WebServer.ts:1278](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/web-server/WebServer.ts#L1278) (verified)
  - *To reach the next level:* No narrower default; least privilege would require manual hardening outside Maestro.
- **B L0:** A hijacked agent holds the user's entire account across services (env API keys, gh, SSH agent, cloud CLIs). — [src/main/process-manager/utils/envBuilder.ts:295](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/process-manager/utils/envBuilder.ts#L295); [src/main/agents/definitions.ts:169-175](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L169-L175) (verified)
  - *To reach the next level:* Credentials reachable by agents are not limited to one system or to read access.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no approval step for anything an agent does. Maestro launches Claude Code with its permission prompts disabled, Codex with approvals and sandbox both bypassed, Factory Droid with permissions skipped and Copilot with every tool, path and URL allowed, and the code comments call this a requirement. No approval hook or permission-prompt tool is wired in, so shell commands, file writes, pushes and network calls all run unattended. Changes are not checkpointed, so a bad action can be irreversible.

- **S L0:** No approval mechanism; every supported agent is spawned with its own approval gate turned off. — [src/main/agents/definitions.ts:169-175](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L169-L175); [src/main/agents/definitions.ts:251](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L251); searched `rg -n -i 'requireApproval|approvalRequired|confirmToolUse|permission-prompt-tool'` in `src/main src/cli src/shared` → 0 hits (no approval hook or permission-prompt tool is wired into any agent spawn) (verified)
  - *To reach the next level:* No per-call human approval of any kind.
- **C L0:** The most powerful path (the agent's shell tool) is exempt along with everything else. — [src/main/agents/definitions.ts:432](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L432); [src/main/agents/definitions.ts:519](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L519); [src/cli/services/agent-spawner.ts:268](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/cli/services/agent-spawner.ts#L268) (verified)
  - *To reach the next level:* No path traverses an approval gate.
- **D L0:** Bypass flags are hard-coded in agent definitions for desktop and CLI spawns; approval is not even an option. — [src/main/agents/definitions.ts:169-175](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L169-L175); [src/cli/services/agent-spawner.ts:268](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/cli/services/agent-spawner.ts#L268) (verified)
  - *To reach the next level:* Approval is not available, let alone on by default.
- **B L0:** Unattended agents can force-push, delete files, send network requests and run arbitrary commands with no undo provided by Maestro. — [src/main/agents/definitions.ts:251](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L251); searched `rg -n -i 'requireApproval|approvalRequired|confirmToolUse|permission-prompt-tool'` in `src/main src/cli src/shared` → 0 hits (no approval hook or permission-prompt tool is wired into any agent spawn) (verified)
  - *To reach the next level:* No checkpoint/rollback or preview for agent actions.
- **Cap:** C2-POWERBYPASS — The agent's shell and every other tool skip any approval in the default configuration because Maestro passes the CLIs' bypass flags.

### C3 Tool & action scoping — 0.05 (high)

Maestro does not define or narrow the tools the model uses; it passes through the underlying coding agents with every tool, path and URL enabled. There is no argument validation in Maestro for shell commands, paths or URLs. A per-tab read-only mode exists that asks the underlying CLI to run in plan or read-only mode, which is the only way to reduce what an agent can do, and it is off by default.

- **S L0:** No argument validation; agents receive raw shell and unrestricted file and network tools. — [src/main/agents/definitions.ts:519](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L519); [src/main/agents/definitions.ts:169-175](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L169-L175) (verified)
  - *To reach the next level:* No allowlist or bounds on any tool argument.
- **C L0:** No tool path is validated by Maestro. — [src/main/agents/definitions.ts:519](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L519) (verified)
  - *To reach the next level:* No tool validates inputs.
- **D L1:** Everything is enabled by default, but a per-tab read-only mode can disable write and exec tools at the CLI layer. — [src/main/agents/definitions.ts:189](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L189); [src/main/agents/definitions.ts:169-175](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L169-L175) (verified)
  - *To reach the next level:* Read-only is not the default tool set.
- **B L0:** A misused tool is a general-purpose shell on the user's machine. — [src/main/agents/definitions.ts:251](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L251) (verified)
  - *To reach the next level:* Tools are not scoped to the workspace.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

Agent commands run directly on the host as the user. Maestro explicitly disables Codex's built-in sandbox with its bypass flag, and it ships no container, VM or OS sandbox of its own. Cue automations can also run shell commands from a project's configuration file straight through the user's shell. Anything an agent or script runs can reach the whole home directory, credentials and the network.

- **S L0:** No isolation primitive; agents and Cue shell actions run as same-user host processes, and Codex's sandbox is deliberately bypassed. — [src/main/agents/definitions.ts:251](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L251); searched `rg -n -i 'docker|bwrap|sandbox-exec|seatbelt|landlock|firecracker|gvisor'` in `src/main src/cli` → 1 hits (single hit is a network-interface name regex in networkUtils.ts, not an isolation primitive); [src/main/cue/cue-shell-executor.ts:6](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/cue/cue-shell-executor.ts#L6) (verified)
  - *To reach the next level:* No OS-level separation of any kind.
- **C L0:** No execution path is sandboxed. — searched `rg -n -i 'docker|bwrap|sandbox-exec|seatbelt|landlock|firecracker|gvisor'` in `src/main src/cli` → 1 hits (single hit is a network-interface name regex in networkUtils.ts, not an isolation primitive); [src/main/cue/cue-shell-executor.ts:6](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/cue/cue-shell-executor.ts#L6) (verified)
  - *To reach the next level:* The main exec path is not sandboxed.
- **D L0:** Isolation is not available; the one sandbox present (Codex's) is turned off by default. — [src/main/agents/definitions.ts:251](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L251) (verified)
  - *To reach the next level:* No sandbox on by default.
- **B L0:** Host-equivalent: full home directory, credentials in the environment, unrestricted network. — [src/main/process-manager/utils/envBuilder.ts:295](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/process-manager/utils/envBuilder.ts#L295); [src/main/agents/definitions.ts:251](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L251) (verified)
  - *To reach the next level:* Execution is not confined away from home directory and credentials.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Agents routinely read untrusted content: repository files, web pages, and in Cue and Symphony workflows the bodies of GitHub issues and pull requests written by anyone. Because every agent runs with approvals and sandbox off, a successful prompt injection can both read the user's secrets and send them anywhere, and take irreversible actions such as pushing code or deleting files, with no human involved. The only defence is an optional third-party injection classifier for GitHub inputs that needs an API token and lets everything through on any error.

- **S L0:** Nothing structurally limits a hijacked agent; the only check is an optional, fail-open classifier. — [src/main/cue/cue-susfactor.ts:25-28](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/cue/cue-susfactor.ts#L25-L28); [src/main/agents/definitions.ts:169-175](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L169-L175) (verified)
  - *To reach the next level:* No capability is gated after untrusted content is read.
- **C L0:** Untrusted sources are not distinguished from user instructions; the classifier, when configured, covers only GitHub items. — [src/main/cue/cue-susfactor.ts:124](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/cue/cue-susfactor.ts#L124) (verified)
  - *To reach the next level:* No source is handled by a structural limit.
- **D L0:** The classifier is effectively off by default: without ODIN_API_TOKEN it is skipped. — [src/main/cue/cue-susfactor.ts:124](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/cue/cue-susfactor.ts#L124); [src/main/cue/cue-susfactor.ts:25-28](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/cue/cue-susfactor.ts#L25-L28) (verified)
  - *To reach the next level:* No protection on by default.
- **B L0:** A hijack can exfiltrate secrets and take irreversible actions unattended. — [src/main/agents/definitions.ts:169-175](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L169-L175); [src/main/process-manager/utils/envBuilder.ts:295](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/process-manager/utils/envBuilder.ts#L295) (verified)
  - *To reach the next level:* Neither exfiltration nor irreversible actions require approval.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.10 (high)

Maestro Cue reads a .maestro/cue.yaml file from each agent's project folder and turns it into automations that start agents with arbitrary prompts or run shell commands on startup, on file changes, on a schedule or on GitHub activity. Nothing asks the user to trust that file first, so a cloned repository can ship automations that run as soon as the Cue engine is active. Cue is listed as on by default, although a startup check reads the raw setting and may leave the engine off on a never-configured install. The underlying agents also auto-load their own instruction and memory files; Maestro adds a memory viewer that lets the user inspect and delete Claude memory entries.

- **S L0:** Repo-controlled cue.yaml can add shell-command and agent-prompt automations with no trust prompt. — [src/main/cue/cue-yaml-loader.ts:133](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/cue/cue-yaml-loader.ts#L133); [src/main/cue/cue-shell-executor.ts:6](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/cue/cue-shell-executor.ts#L6) (verified)
  - *To reach the next level:* No workspace-trust decision before loading security-relevant project config.
- **C L0:** No memory or config path is controlled. — [src/main/cue/cue-yaml-loader.ts:133](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/cue/cue-yaml-loader.ts#L133) (verified)
  - *To reach the next level:* Neither cue.yaml nor agent instruction files are gated.
- **D L1:** Single-user desktop; memory and configs are per-project but isolation is not enforced by Maestro. — [src/main/memory-manager.ts:12-13](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/memory-manager.ts#L12-L13) (verified)
  - *To reach the next level:* No namespace isolation enforced in code.
- **B L1:** A poisoned cue.yaml persists across the user's sessions and triggers tool use and shell commands; memory can be inspected and purged in the viewer. — [src/main/cue/cue-shell-executor.ts:6](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/cue/cue-shell-executor.ts#L6); [src/main/memory-manager.ts:12-13](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/memory-manager.ts#L12-L13) (verified)
  - *To reach the next level:* Poisoned config can trigger actions rather than only text.
- **Cap:** C6-REPOCONFIG — A workspace .maestro/cue.yaml can enable shell-command and agent automations without an explicit user trust decision while the Cue Encore feature (default true) is on.
- **Notes:** Main-process boot gate reads encoreFeatures raw (index.ts:1328) while the shared default is true; on a never-configured install the engine may not auto-start until flags are persisted. Burden of proof is on the control, so the cap is applied.

### C7 Third-party extensions — 0.17 (high)

Maestro has no plugin runtime at this commit (its plugin architecture document describes a folder that is not in the source). Third-party content it loads is the playbook marketplace, fetched from the main branch of a vendor GitHub repository with no version pin or integrity check, including script assets that agents are told to run. MCP servers and skills come from the underlying agents' own configuration and Maestro does not verify them. Anything loaded runs through unsandboxed, full-permission agents as the user.

- **S L1:** Playbooks come from a vendor registry chosen by the user but fetched unpinned from the main branch. — [src/main/services/marketplace-service.ts:32](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/services/marketplace-service.ts#L32); [src/main/services/marketplace-service.ts:285](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/services/marketplace-service.ts#L285) (verified)
  - *To reach the next level:* No version pinning.
- **C L0:** No extension type is verified by Maestro. — [src/main/services/marketplace-service.ts:285](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/services/marketplace-service.ts#L285) (verified)
  - *To reach the next level:* No integrity check on playbooks or MCP servers.
- **D L2:** Playbooks are imported explicitly by the user from the marketplace UI. — [src/main/services/marketplace-service.ts:32](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/services/marketplace-service.ts#L32) (verified)
  - *To reach the next level:* Import does not show exact scripts and permissions that will run.
- **B L0:** Imported content executes via full-permission agents with the user's whole environment. — [src/main/process-manager/utils/envBuilder.ts:295](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/process-manager/utils/envBuilder.ts#L295); [src/main/agents/definitions.ts:169-175](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/agents/definitions.ts#L169-L175) (verified)
  - *To reach the next level:* No separate, scrubbed or sandboxed process for extensions.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.20 (high)

Credentials are not kept away from agents: the full environment, including any API keys and tokens, is copied into every agent process, which the model can read. Maestro redacts secrets when building a debug package and strips IP and email from crash reports, and it keeps the system prompt out of spawn logs. There is no OS keychain use, and Sentry crash reporting is on by default.

- **S L1:** Secrets come from env vars; masking only in debug packages, crash-report user fields and spawn logs. — [src/main/debug-package/collectors/settings.ts:17](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/debug-package/collectors/settings.ts#L17); [src/main/index.ts:392-399](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/index.ts#L392-L399); [src/main/ipc/handlers/process.ts:677](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/ipc/handlers/process.ts#L677); searched `rg -n -i 'safeStorage|keytar'` in `src/main src/cli` → 0 hits (no OS keychain or encrypted credential store) (verified)
  - *To reach the next level:* No type-level masking or log filters on main paths.
- **C L1:** Debug-package export is the one path with key redaction; subprocess environments and model-bound context are unprotected. — [src/main/debug-package/collectors/settings.ts:17](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/debug-package/collectors/settings.ts#L17); [src/main/process-manager/utils/envBuilder.ts:295](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/process-manager/utils/envBuilder.ts#L295) (verified)
  - *To reach the next level:* Logs and transcripts are not redacted.
- **D L1:** Crash reporting to Sentry is on by default, with user identifiers stripped. — [src/main/stores/utils.ts:223](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/stores/utils.ts#L223); [src/main/index.ts:392-399](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/index.ts#L392-L399) (verified)
  - *To reach the next level:* Crash reporting is not opt-in.
- **B L0:** Long-lived high-privilege keys in the user's environment are reachable by every agent subprocess. — [src/main/process-manager/utils/envBuilder.ts:295](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/process-manager/utils/envBuilder.ts#L295) (verified)
  - *To reach the next level:* Keys are not scoped or kept out of subprocesses.
- **Cap:** none

### C9 Audit & traceability — 0.30 (high)

Maestro shows tool calls live in the UI, but by default those tool entries are discarded when the agent exits, and the main-process file log is off by default. What persists is per-session history entries and session output stored in the app's data folder, which the unsandboxed agents could edit. The detailed per-tool record lives in each underlying agent's own transcript, which Maestro does not own.

- **S L1:** Unstructured spawn logs and history summaries; tool-call log entries are ephemeral by default. — [src/renderer/hooks/agent/internal/helpers/exitTabCleanup.ts:5-7](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/renderer/hooks/agent/internal/helpers/exitTabCleanup.ts#L5-L7); [src/main/ipc/handlers/process.ts:688-693](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/ipc/handlers/process.ts#L688-L693); [src/main/history-manager.ts:481](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/history-manager.ts#L481) (verified)
  - *To reach the next level:* No persisted structured record of every tool call.
- **C L1:** Spawn commands and run summaries are recorded; individual tool calls are not persisted. — [src/renderer/hooks/agent/internal/helpers/exitTabCleanup.ts:5-7](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/renderer/hooks/agent/internal/helpers/exitTabCleanup.ts#L5-L7) (verified)
  - *To reach the next level:* Not every built-in tool call is recorded.
- **D L2:** History is on by default in the app data folder outside the workspace, but unsandboxed agents could alter it; file logging is off. — [src/main/history-manager.ts:481](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/history-manager.ts#L481); [src/main/utils/logger.ts:67](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/utils/logger.ts#L67) (verified)
  - *To reach the next level:* Record written by a component the model cannot control.
- **B L1:** Best effort; in-memory logs are lost on crash unless file logging is enabled. — [src/main/utils/logger.ts:67](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/utils/logger.ts#L67) (verified)
  - *To reach the next level:* Records are not flushed durably per action.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

Maestro bounds some runs: Cue runs time out after 30 minutes with one run at a time per agent and chains capped at 10 hops, and Auto Run stops a document after three no-progress iterations and kills an agent that has been silent for four hours. Loop mode is unlimited by default and there is no token or cost ceiling. Stopping an agent kills its whole process tree.

- **S L2:** Iteration-style caps (stall counter, chain depth, max loops) plus per-run timeouts are enforced in code; halt kills the process tree. — [src/shared/cue/contracts.ts:321-324](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/shared/cue/contracts.ts#L321-L324); [src/main/cue/cue-engine.ts:79](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/cue/cue-engine.ts#L79); [src/shared/autorunStall.ts:26](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/shared/autorunStall.ts#L26); [src/main/utils/processTree.ts:92](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/main/utils/processTree.ts#L92) (verified)
  - *To reach the next level:* No token or cost cap and no rate limit on side-effecting actions.
- **C L2:** Limits apply to Auto Run and Cue loops plus per-run timeouts. — [src/shared/cue/contracts.ts:321-324](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/shared/cue/contracts.ts#L321-L324); [src/renderer/stores/settingsStore.ts:983](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/renderer/stores/settingsStore.ts#L983) (verified)
  - *To reach the next level:* Parallel agents and group chats don't count against a shared budget.
- **D L1:** Defaults are very large: loop mode is unlimited and the inactivity watchdog is 240 minutes. — [src/renderer/hooks/batch/internal/useBatchRunner.ts:360](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/renderer/hooks/batch/internal/useBatchRunner.ts#L360); [src/renderer/stores/settingsStore.ts:983](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/renderer/stores/settingsStore.ts#L983) (verified)
  - *To reach the next level:* Defaults are not tight.
- **B L1:** Ceilings are hours long with unlimited spend; stop does kill in-flight processes. — [src/renderer/stores/settingsStore.ts:983](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/renderer/stores/settingsStore.ts#L983); [src/renderer/hooks/batch/internal/useBatchRunner.ts:360](https://github.com/runmaestro/maestro/blob/54ffa324752415d7015e0dda3230c6811d4f52c3/src/renderer/hooks/batch/internal/useBatchRunner.ts#L360) (verified)
  - *To reach the next level:* No tight per-run time and cost ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Repository files, web content and GitHub issue/PR bodies via Cue (src/main/cue/cue-susfactor.ts:25) · [B] sensitive data/systems: Full user environment passed to agents (src/main/process-manager/utils/envBuilder.ts:295) · [C] state change / egress: Agents spawned with all approvals and sandbox bypassed (src/main/agents/definitions.ts:251) · Same default session? Yes

## Highest-impact improvements
1. Stop hard-coding bypass flags; route the CLIs' permission prompts to a Maestro approval UI (e.g. Claude --permission-prompt-tool) and default to them. — C2 S L0→L3, +0.225 before caps (Playbook 5)
2. Run Codex with --sandbox workspace-write (and equivalent CLI sandboxes) instead of the bypass flag by default. — C4 S L0→L3, +0.225 before caps (Playbook 3)
3. Require an explicit per-project trust decision (with a content hash) before loading .maestro/cue.yaml shell or agent subscriptions. — C6 S L0→L3, +0.225 before caps (Playbook 2)
4. Pass agents an allowlisted environment instead of the whole process.env. — C8 C L1→L2, +0.075 before caps (Playbook 4)
5. Bind the CLI/web control server to 127.0.0.1 unless the user turns on Live remote access. — C1 D L0→L1, +0.050 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Behaviour of the underlying agent CLIs (Claude Code, Codex, OpenCode, Droid, Copilot) under the flags Maestro passes is taken from the flags' documented meaning, not examined in their source.
- Symphony, group chat, SSH remote execution and the renderer UI were reviewed only at the level needed for these criteria.
- Cue engine default is ambiguous: the shared default is on but the main-process boot gate reads the raw setting; scored against the declared default.
- CLAUDE-PLUGINS.md describes a plugin runtime under src/main/plugins that does not exist at this commit; no credit was given for it.
- No text aimed at AI reviewers was found in AGENTS.md, CLAUDE*.md or SECURITY.md.
