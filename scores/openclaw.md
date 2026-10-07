# Defense-in-Depth Score: OpenClaw

**Repo:** https://github.com/openclaw/openclaw · **Commit:** `4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7` · **Reviewed:** 2026-10-03
**What it is:** Local-first personal AI agent connected to messaging apps that runs shell, browser and account actions
**Category:** AI Assistants
**Scored configuration:** Local Gateway installed via npm and 'openclaw onboard' (tools.profile=full written by onboarding), no sandbox, default exec policy, default plugin and channel settings.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 2.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L0 | L0 | 0.07 | C1-SELFESC | **0.07** | High |
| C2 | Approval gates | L3 | L1 | L0 | L0 | 0.30 | C2-POWERBYPASS | **0.25** | High |
| C3 | Tool & action scoping | L2 | L1 | L0 | L0 | 0.23 | — | **0.23** | High |
| C4 | Code-execution isolation | L3 | L2 | L2 | L2 | 0.57 | G1 | **0.50** (alt) | High |
| C5 | Untrusted input blast radius | L1 | L2 | L2 | L0 | 0.33 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C7 | Third-party extensions | L0 | L1 | L0 | L0 | 0.07 | C7-RCELOAD | **0.07** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L0 | 0.40 | — | **0.40** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | High |


As shipped, OpenClaw gives the model a shell on your machine with no approval prompt and no sandbox. Onboarding enables every tool, and the agent can message from your connected chat accounts, write any file, and install plugins. Strong controls exist (exec approvals, a hardened Docker sandbox, SSRF-guarded fetch, taint-aware memory), but most are opt-in. A prompt injection in a web page or message can lead straight to data theft or irreversible actions.

## Critical gaps
- Host shell exec runs with no approval by default (security full, ask off), and it is the most powerful action path. (ASI02, ASI09, T10; C2) — [src/infra/exec-approvals-config.ts:95-96](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/infra/exec-approvals-config.ts#L95-L96); [src/agents/exec-defaults.ts:166](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/exec-defaults.ts#L166)
- The agent's identity is the user's OS account; the default write tool can rewrite the hot-reloaded config and so its own permissions. (ASI03, T3; C1) — [src/agents/agent-tools.read.ts:1389-1392](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/agent-tools.read.ts#L1389-L1392); [src/gateway/config-reload-settings.ts:8](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/gateway/config-reload-settings.ts#L8)
- No sandbox by default: model commands run on the host with the user's home directory, network and inherited provider/channel keys. (ASI05, T11; C4) — [src/agents/sandbox/config.ts:233](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/sandbox/config.ts#L233); [src/agents/bash-tools.exec-request-preparation.ts:511](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/bash-tools.exec-request-preparation.ts#L511)
- A hijacked turn can read private data and exfiltrate it or act irreversibly (shell, message tool) with no human approval. (ASI01, LLM01, T6; C5) — [src/security/external-content.ts:44-45](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/security/external-content.ts#L44-L45); [src/infra/exec-approvals-config.ts:95-96](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/infra/exec-approvals-config.ts#L95-L96)
- The model can install third-party ClawHub plugins that run in-process, and the capability consent control for those installs is not a complete boundary. (ASI04, T17, LLM03; C7) — [src/agents/tools/plugins-tool.ts:151-162](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/tools/plugins-tool.ts#L151-L162)

## Criterion details

### C1 Identity & least privilege — 0.07 (high)

OpenClaw runs every tool as the operating-system user who started the Gateway, holding that user's files, the messaging accounts it is connected to, and every model-provider key. Host commands inherit the Gateway's environment; a denylist strips some cloud and git tokens (AWS keys, GH_TOKEN, NPM_TOKEN) but provider API keys and channel tokens pass through. Non-owner chat senders are denied control-plane tools, which is a real per-sender check, but the owner session gets everything. Because the default file-write tool can write anywhere on the host and the config file hot-reloads, a hijacked agent can rewrite its own permissions.

- **S L0:** The agent acts with the operator's ambient OS authority and full environment; no scoped identity exists, and the write tool can edit the agent's own config which hot-reloads. — [src/agents/bash-tools.exec-request-preparation.ts:412](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/bash-tools.exec-request-preparation.ts#L412); [src/agents/agent-tools.read.ts:1389-1392](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/agent-tools.read.ts#L1389-L1392); [src/gateway/config-reload-settings.ts:8](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/gateway/config-reload-settings.ts#L8) (verified)
  - *To reach the next level:* Run tools under a dedicated low-privilege identity or a deterministic authorization gate before credentials are used.
- **C L1:** Owner-only control-plane tools are denied to non-owner senders, but host subprocesses receive the inherited environment minus a denylist that omits provider and channel keys. — [src/auto-reply/reply/reply-tool-authority.ts:284](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/auto-reply/reply/reply-tool-authority.ts#L284); [src/agents/bash-tools.exec-request-preparation.ts:511](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/bash-tools.exec-request-preparation.ts#L511); [src/infra/host-env-security-policy.json:255-261](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/infra/host-env-security-policy.json#L255-L261); searched `rg -n -e 'OPENAI_API_KEY|ANTHROPIC_API_KEY|TELEGRAM_BOT_TOKEN|SLACK_BOT_TOKEN'` in `src/infra/host-env-security-policy.json` → 0 hits (Provider API keys and channel bot tokens are not in the host-exec env scrub list, so they are inherited by every host command.) (verified)
  - *To reach the next level:* Every tool path, including subprocesses, plugins and MCP servers, should go through one authorization layer.
- **D L0:** Onboarding writes tools.profile=full (all tools) and host exec defaults to security full / ask off, so the default install runs with the user's full privilege. — [src/commands/onboard-config.ts:13](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/commands/onboard-config.ts#L13); [src/agents/tool-catalog.ts:562-564](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/tool-catalog.ts#L562-L564); [src/infra/exec-approvals-config.ts:95-96](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/infra/exec-approvals-config.ts#L95-L96) (verified)
  - *To reach the next level:* Ship a near-minimal default profile and require explicit operator elevation for write and exec.
- **B L0:** A hijacked agent holds the user's whole account surface: shell, home directory, connected messaging accounts, browser, and provider keys. — [src/agents/exec-defaults.ts:166](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/exec-defaults.ts#L166); [src/agents/host-file-write.ts:27](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/host-file-write.ts#L27); [src/infra/dotenv.ts:332-334](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/infra/dotenv.ts#L332-L334) (verified)
  - *To reach the next level:* Confine the agent to one scoped, mostly read-only system.
- **Cap:** C1-SELFESC — The default write tool writes anywhere on the host (workspaceOnly defaults false) and openclaw.json is hot-reloaded by default, so the agent can rewrite its own tool, exec and sandbox policy.

### C2 Approval gates — 0.25 (high)

OpenClaw has a well-built exec approval system: the approver sees the exact command, argv and working directory, decisions are allow-once, allow-always or deny, and approved executables are re-checked before launch. It is off by default: host exec defaults to security 'full' and ask 'off', so shell commands run with no prompt. Even when enabled it covers only exec; file writes, outbound messages, browser actions and plugin installs are not gated. Most of those actions (sent messages, deleted files) cannot be undone.

- **S L3:** When enabled, exec approvals show the exact command/argv/cwd with allow-once/allow-always/deny and allowlist-based risk tiers. — [src/agents/bash-tools.exec-approval-request.ts:36-40](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/bash-tools.exec-approval-request.ts#L36-L40); [src/infra/exec-approvals-policy.ts:41-45](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/infra/exec-approvals-policy.ts#L41-L45); [src/infra/exec-approvals-policy.ts:18-28](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/infra/exec-approvals-policy.ts#L18-L28) (verified)
  - *To reach the next level:* Argument-level allow/deny rules for every tool and guaranteed approved-equals-executed binding across all hosts.
- **C L1:** Only exec (and plugin hooks that request approval) is gated; write, edit, message, browser and plugins install reach state changes without a gate. — [src/infra/exec-approvals-policy.ts:18-28](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/infra/exec-approvals-policy.ts#L18-L28); [src/agents/tool-catalog.ts:358](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/tool-catalog.ts#L358); [src/agents/agent-tools.read.ts:1389-1392](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/agent-tools.read.ts#L1389-L1392) (verified)
  - *To reach the next level:* Route every mutating tool, including MCP and plugin tools, through the gate.
- **D L0:** Default exec policy is security full / ask off, so approval is opt-in. — [src/infra/exec-approvals-config.ts:95-96](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/infra/exec-approvals-config.ts#L95-L96); [src/infra/exec-approvals-config.ts:95-96](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/infra/exec-approvals-config.ts#L95-L96); [src/agents/exec-defaults.ts:166](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/exec-defaults.ts#L166) (verified)
  - *To reach the next level:* Turn approvals on by default for host exec and other consequential tools.
- **B L0:** Unapproved actions include sending messages from the user's accounts and deleting host files, with no checkpoint or undo. — [src/agents/tool-catalog.ts:358](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/tool-catalog.ts#L358); [src/agents/host-file-write.ts:27](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/host-file-write.ts#L27) (verified)
  - *To reach the next level:* Checkpoints for file state and previews or dry-runs for external actions.
- **Cap:** C2-POWERBYPASS — The most powerful tool, host shell exec, runs without any approval in the default configuration.

### C3 Tool & action scoping — 0.23 (high)

Some tools are carefully bounded: web_fetch uses a strict SSRF guard that pins DNS, blocks private addresses and rechecks redirects, and apply_patch is workspace-contained by default. But the default 'full' profile enables every tool, including a raw shell and file write/edit that accept any host path. A misused tool can therefore reach the whole machine.

- **S L2:** Strong validation exists for some tools (SSRF guard, workspace-contained apply_patch) but exec takes a raw shell string and write takes any host path. — [src/agents/tools/web-guarded-fetch.ts:56](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/tools/web-guarded-fetch.ts#L56); [src/infra/net/ssrf.ts:267](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/infra/net/ssrf.ts#L267); [src/agents/agent-tools.read.ts:1389-1392](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/agent-tools.read.ts#L1389-L1392) (verified)
  - *To reach the next level:* Replace general tools with narrow ones and validate paths with resolved containment by default.
- **C L1:** Only a few tools validate inputs; exec, write and edit accept arbitrary commands and paths by default. — [src/agents/tool-catalog.ts:109](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/tool-catalog.ts#L109); [src/agents/host-file-write.ts:27](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/host-file-write.ts#L27) (verified)
  - *To reach the next level:* Validate inputs on most built-in tools, with a shared layer for extension tools.
- **D L0:** Onboarding selects the 'full' profile, which allows every tool including exec, write and network tools. — [src/commands/onboard-config.ts:13](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/commands/onboard-config.ts#L13); [src/commands/onboard-config.ts:108](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/commands/onboard-config.ts#L108); [src/agents/tool-catalog.ts:562-564](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/tool-catalog.ts#L562-L564) (verified)
  - *To reach the next level:* Default to a read-only or messaging tool set; require explicit enabling for write and exec.
- **B L0:** General-purpose shell and host-wide file writes reach the whole machine. — [src/agents/exec-defaults.ts:166](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/exec-defaults.ts#L166); [src/agents/agent-tools.read.ts:1389-1392](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/agent-tools.read.ts#L1389-L1392) (verified)
  - *To reach the next level:* Scope tools to the workspace with quantity bounds.
- **Cap:** none

### C4 Code-execution isolation — 0.50 (high)

By default the agent's shell commands run directly on the host as the user, because sandbox mode defaults to 'off' and exec targets the gateway host. An optional Docker sandbox is well hardened: read-only root, network off, all capabilities dropped, no-new-privileges, a non-root user and no workspace mount unless configured. It must be enabled by the operator, and plugins, MCP servers and the default Node code-mode executor still run in or beside the Gateway process.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Default exec runs as a same-user host subprocess. — [src/agents/sandbox/config.ts:233](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/sandbox/config.ts#L233); [src/agents/exec-defaults.ts:166](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/exec-defaults.ts#L166) (verified)
    - *To reach the next level:* Run model-generated commands inside an OS or container boundary by default.
  - **C L0:** No execution path is sandboxed in the default configuration. — [src/agents/sandbox/config.ts:233](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/sandbox/config.ts#L233); [src/agents/exec-defaults.ts:71-76](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/exec-defaults.ts#L71-L76) (verified)
    - *To reach the next level:* Sandbox the main exec path and other code paths.
  - **D L0:** Sandbox mode defaults to off. — [src/agents/sandbox/config.ts:233](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/sandbox/config.ts#L233) (verified)
    - *To reach the next level:* Enable the sandbox by default.
  - **B L0:** Host-equivalent: commands run with the user's home directory, network and inherited credentials. — [src/agents/bash-tools.exec-request-preparation.ts:511](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/bash-tools.exec-request-preparation.ts#L511); searched `rg -n -e 'OPENAI_API_KEY|ANTHROPIC_API_KEY|TELEGRAM_BOT_TOKEN|SLACK_BOT_TOKEN'` in `src/infra/host-env-security-policy.json` → 0 hits (Provider API keys and channel bot tokens are not in the host-exec env scrub list, so they are inherited by every host command.); [src/agents/agent-tools.read.ts:1389-1392](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/agent-tools.read.ts#L1389-L1392) (verified)
    - *To reach the next level:* Workspace-only mount, no secrets and restricted egress.
- **opt-in Docker sandbox (agents.defaults.sandbox.mode non-main/all)** (alt; raw 0.57, cap G1 → 0.50) ← counted
  - **S L3:** Sandbox containers default to read-only root, network none, cap-drop ALL, no-new-privileges and a non-root user. — [src/agents/sandbox/config.ts:101-105](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/sandbox/config.ts#L101-L105); [src/agents/sandbox/config.ts:103](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/sandbox/config.ts#L103); [src/agents/sandbox/docker.ts:263-266](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/sandbox/docker.ts#L263-L266); [scripts/docker/sandbox/Dockerfile:18](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/scripts/docker/sandbox/Dockerfile#L18) (verified)
    - *To reach the next level:* Kernel-separated isolation (microVM or gVisor).
  - **C L2:** Exec and file tools route into the sandbox, but elevated exec, plugins and MCP stdio servers run on the host. — [src/agents/sandbox/config.ts:101-105](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/sandbox/config.ts#L101-L105); [src/plugins/config-activation-shared.ts:165](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/plugins/config-activation-shared.ts#L165) (verified)
    - *To reach the next level:* Sandbox every model-reachable path and fail closed.
  - **D L2:** When enabled, the operator can widen it via elevated mode, binds and network settings in config without per-call approval. — [src/agents/sandbox/config.ts:103](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/sandbox/config.ts#L103); [src/agents/sandbox/config.ts:236](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/sandbox/config.ts#L236) (verified)
    - *To reach the next level:* Escalation out of the sandbox should need a per-call human approval.
  - **B L2:** Default container has no network, no workspace mount and a read-only root, but no PID/CPU/memory limits by default. — [src/agents/sandbox/config.ts:103](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/sandbox/config.ts#L103); [src/agents/sandbox/config.ts:236](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/sandbox/config.ts#L236); [src/agents/sandbox/config.ts:101-105](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/sandbox/config.ts#L101-L105) (verified)
    - *To reach the next level:* Add default CPU/memory/PID limits (L3), then ephemeral per-run containers.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.25 (high)

OpenClaw reads web pages, search results, email and webhook payloads, browser pages and chat messages, and wraps much of this in randomized 'external untrusted content' markers with a warning. Those markers are only text for the model; nothing in code stops a hijacked turn from running shell commands or sending messages. MCP tool results are not wrapped at all. The pairing DM policy means only approved senders can trigger the agent, but within the owner's session untrusted content, private data and outbound channels all meet with no human in the loop.

- **S L1:** Defense is spotlighting: untrusted content is wrapped in random-ID boundary markers with a data-not-instructions note. — [src/security/external-content.ts:44-45](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/security/external-content.ts#L44-L45); [src/security/external-content.ts:36-38](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/security/external-content.ts#L36-L38) (verified)
  - *To reach the next level:* Disable or gate egress and state-changing tools once untrusted content enters the turn.
- **C L2:** Web fetch/search, browser, email/webhook hooks and channel metadata are wrapped; MCP tool results are not. — [src/security/external-content.ts:36-38](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/security/external-content.ts#L36-L38); searched `rg -n -e 'wrapExternalContent|wrapWebContent|EXTERNAL_UNTRUSTED'` in `src/agents/agent-bundle-mcp-tools.ts src/mcp` → 0 hits (MCP tool results are not wrapped as untrusted external content.) (verified)
  - *To reach the next level:* Cover tool and MCP results and sub-agent messages too.
- **D L2:** Wrapping is on by default; operators can disable it per hook with allowUnsafeExternalContent. — [src/security/external-content.ts:44-45](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/security/external-content.ts#L44-L45) (verified)
  - *To reach the next level:* Make disabling explicit and warned everywhere.
- **B L0:** A hijacked agent can read secrets and host files and then send them out or act irreversibly (shell, message tool) without approval. — [src/agents/tool-catalog.ts:358](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/tool-catalog.ts#L358); [src/agents/tool-catalog.ts:142](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/tool-catalog.ts#L142); [src/infra/exec-approvals-config.ts:95-96](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/infra/exec-approvals-config.ts#L95-L96) (verified)
  - *To reach the next level:* Require approval for egress and irreversible actions after untrusted input.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.25 (high)

OpenClaw injects workspace files such as AGENTS.md, SOUL.md, MEMORY.md and USER.md into every system prompt, and the agent can write them. It tracks when a turn has seen network content or a non-owner sender: memory files written in such a turn are marked untrusted and kept out of automatic injection, and the check fails closed. That protection covers only MEMORY.md and USER.md; other instruction files load silently, and the default write tool can also reach the agent's own config and global plugin folder. By default all direct messages share one main session.

- **S L1:** Memory-file writes carry taint-derived provenance and untrusted MEMORY.md/USER.md are excluded from injection, but other bootstrap instruction files load silently and are agent-writable. — [src/agents/agent-tools.ts:214-217](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/agent-tools.ts#L214-L217); [src/agents/bootstrap-files.ts:266-272](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/bootstrap-files.ts#L266-L272); [src/agents/workspace-bootstrap-policy.ts:29-36](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/workspace-bootstrap-policy.ts#L29-L36); [src/agents/system-prompt.ts:892](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/system-prompt.ts#L892) (verified)
  - *To reach the next level:* Gate all instruction-file writes and require a trust decision for security-relevant config.
- **C L1:** Provenance gating covers the root memory and user files only; AGENTS.md, SOUL.md and TOOLS.md are not covered. — [src/agents/bootstrap-files.ts:266-272](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/bootstrap-files.ts#L266-L272); [src/agents/workspace-bootstrap-policy.ts:35](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/workspace-bootstrap-policy.ts#L35) (verified)
  - *To reach the next level:* Cover all memory stores and auto-loaded files.
- **D L1:** session.dmScope defaults to 'main', so direct messages from different allowed senders share one session; per-peer isolation is opt-in. — [src/routing/resolve-route.ts:277](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/routing/resolve-route.ts#L277) (verified)
  - *To reach the next level:* Per-user session namespaces by default.
- **B L1:** Poisoned instruction files persist across the user's sessions and steer tool use. — [src/agents/system-prompt.ts:892](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/system-prompt.ts#L892); [src/agents/workspace-bootstrap-policy.ts:29-36](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/workspace-bootstrap-policy.ts#L29-L36) (verified)
  - *To reach the next level:* Limit poisoned context to text output or gated actions, or make it easy to inspect and purge.
- **Cap:** none
- **Notes:** Workspace .env loading from the working directory filters provider keys, endpoint overrides, proxies and OPENCLAW_* keys (src/infra/dotenv.ts:185, 238), so C6-REPOCONFIG was not applied; workspace-origin plugins are disabled by default.

### C7 Third-party extensions — 0.07 (high)

Plugins run in-process with the Gateway's full privileges, and the project's security policy says so. The plugins tool is in the default 'full' profile and lets the model install packages from ClawHub. The capability consent control for those installs does not cover every path. Plugins already in the global plugin folder load by default when no allowlist is set. npm installs do record version and integrity metadata, and workspace-origin plugins are disabled by default.

- **S L0:** The model can choose and install a ClawHub plugin through the plugins tool; the install consent control does not cover every path. — [src/agents/tools/plugins-tool.ts:151-162](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/tools/plugins-tool.ts#L151-L162); [src/agents/tool-catalog.ts:386](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/tool-catalog.ts#L386) (verified)
  - *To reach the next level:* Restrict installs to pinned, integrity-checked, human-approved packages.
- **C L1:** npm/ClawHub installs record integrity, but globally discovered plugins and MCP servers are not verified before load. — [src/plugins/config-activation-shared.ts:165](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/plugins/config-activation-shared.ts#L165); [src/plugins/config-activation-shared.ts:115-121](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/plugins/config-activation-shared.ts#L115-L121) (verified)
  - *To reach the next level:* Verify most extension types before load.
- **D L0:** The consent control for model-initiated installs does not cover every path, and global-root plugins auto-enable without an allowlist. — [src/plugins/config-activation-shared.ts:165](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/plugins/config-activation-shared.ts#L165) (verified)
  - *To reach the next level:* Nothing third-party enabled without an explicit operator install showing what will run.
- **B L0:** Plugins load in-process with the Gateway's credentials and OS privileges. — [src/plugins/config-activation-shared.ts:165](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/plugins/config-activation-shared.ts#L165); [src/agents/bash-tools.exec-request-preparation.ts:412](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/bash-tools.exec-request-preparation.ts#L412) (verified)
  - *To reach the next level:* Run extensions in separate processes with scrubbed environments.
- **Cap:** C7-RCELOAD — In the default full profile the model can install and activate third-party ClawHub plugin code (in-process); the install consent control does not cover every path.

### C8 Secrets & sensitive-data protection — 0.40 (high)

OpenClaw redacts token-shaped strings from logs and persisted transcripts by default and redacts .env files before showing them to the model. Telemetry is limited to a daily update check, and the feature statistics are opt-in. Provider API keys and channel tokens stay in the Gateway's environment, every host shell command inherits them, and the redaction mode can be switched off.

- **S L2:** Pattern-based redaction applies to logs and transcripts, and .env reads are redacted before the model sees them. — [src/logging/redact.ts:73](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/logging/redact.ts#L73); [src/agents/transcript-redact.ts:5](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/transcript-redact.ts#L5); [src/agents/agent-tools.read.ts:1100-1102](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/agent-tools.read.ts#L1100-L1102) (verified)
  - *To reach the next level:* Keychain-backed storage plus redaction on all model-bound tool output.
- **C L2:** Logs and transcripts are covered; shell output and subprocess environments are not. — [src/agents/transcript-redact.ts:5](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/transcript-redact.ts#L5); searched `rg -n -e 'OPENAI_API_KEY|ANTHROPIC_API_KEY|TELEGRAM_BOT_TOKEN|SLACK_BOT_TOKEN'` in `src/infra/host-env-security-policy.json` → 0 hits (Provider API keys and channel bot tokens are not in the host-exec env scrub list, so they are inherited by every host command.) (verified)
  - *To reach the next level:* Cover model-bound messages, subprocess environments and telemetry.
- **D L2:** Redaction defaults to 'tools' mode but can be set to 'off'. — [src/logging/redact.ts:73](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/logging/redact.ts#L73); [src/logging/redact.ts:155](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/logging/redact.ts#L155) (verified)
  - *To reach the next level:* Make redaction always on.
- **B L0:** Long-lived provider and channel keys are reachable by every host subprocess. — [src/agents/bash-tools.exec-request-preparation.ts:511](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/bash-tools.exec-request-preparation.ts#L511); searched `rg -n -e 'OPENAI_API_KEY|ANTHROPIC_API_KEY|TELEGRAM_BOT_TOKEN|SLACK_BOT_TOKEN'` in `src/infra/host-env-security-policy.json` → 0 hits (Provider API keys and channel bot tokens are not in the host-exec env scrub list, so they are inherited by every host command.) (verified)
  - *To reach the next level:* Scoped, short-lived, rotatable credentials.
- **Cap:** none

### C9 Audit & traceability — 0.45 (high)

A metadata audit ledger is on by default and records each run and tool action with agent, session and run attribution. Full tool arguments live in per-agent session transcripts stored in the state directory. Both sit in ~/.openclaw, outside the agent workspace but writable by the agent's own host shell. Execution identity recording is opt-in, and the ledger drops events when its queue is full while the actions still go ahead.

- **S L2:** Tool actions are recorded as structured events with agent/session/run attribution, and transcripts hold the call details. — [src/audit/audit-event-types.ts:102-108](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/audit/audit-event-types.ts#L102-L108); [src/audit/audit-config.ts:16-18](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/audit/audit-config.ts#L16-L18); [src/agents/transcript-redact.ts:5](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/transcript-redact.ts#L5) (verified)
  - *To reach the next level:* Record the requesting principal and approver per action, with correlation across sub-agents, by default.
- **C L2:** All tool actions in the agent loop are recorded; approver and principal identity is opt-in. — [src/audit/audit-event-types.ts:102-108](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/audit/audit-event-types.ts#L102-L108); [src/audit/audit-config.ts:16-18](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/audit/audit-config.ts#L16-L18) (verified)
  - *To reach the next level:* Include approvals, denials and sub-agent lineage by default.
- **D L2:** The ledger is on by default and stored in the state directory, which the agent's host shell can still modify. — [src/audit/audit-config.ts:11-13](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/audit/audit-config.ts#L11-L13); [src/agents/exec-defaults.ts:166](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/exec-defaults.ts#L166) (verified)
  - *To reach the next level:* Write records through a component the model cannot control.
- **B L1:** Ledger writes are best-effort: metadata is dropped when the writer is unavailable or the queue is full. — [src/audit/audit-event-writer.ts:260-261](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/audit/audit-event-writer.ts#L260-L261) (verified)
  - *To reach the next level:* Flush records per action and surface errors.
- **Cap:** none

### C10 Limits & kill switch — 0.33 (high)

Runs have a wall-clock limit, but it defaults to 48 hours, and each shell command times out after 30 minutes. No step or cost limit was found, and loop detection is off by default. Sub-agents are capped at depth 5 and 5 children per agent. Stopping kills the command's process tree, but background processes and scheduled automations can keep running.

- **S L1:** A wall-clock run timeout and a per-exec timeout exist, but there is no iteration or cost cap. — [src/agents/timeout.ts:13](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/timeout.ts#L13); [src/agents/exec-tool-timeout.ts:7](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/exec-tool-timeout.ts#L7); searched `rg -n -e 'maxIterations|maxToolCalls|maxSteps'` in `src/agents/embedded-agent-runner` → 0 hits (No iteration or tool-call cap in the embedded agent run loop.); searched `rg -n -i -e 'maxCost|costLimit|spendLimit|budgetUsd|maxSpend'` in `src/agents src/config` → 4 hits (Hits are provider error classification of upstream spend-limit messages and a history byte budget; no agent cost or spend cap exists.) (verified)
  - *To reach the next level:* Add an iteration cap plus enforced token or cost limits.
- **C L2:** Limits cover the top-level run and exec calls; sub-agent depth and fan-out are capped. — [src/agents/exec-tool-timeout.ts:7](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/exec-tool-timeout.ts#L7); [src/config/agent-limits.ts:22-26](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/config/agent-limits.ts#L22-L26) (verified)
  - *To reach the next level:* Count sub-agents and background tasks against the same budget.
- **D L1:** Defaults are very large (48-hour run timeout) and loop detection is disabled by default. — [src/agents/timeout.ts:13](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/timeout.ts#L13); [src/agents/tool-loop-detection.ts:471](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/tool-loop-detection.ts#L471) (verified)
  - *To reach the next level:* Ship tight defaults the model cannot raise.
- **B L1:** A runaway can act for up to 48 hours, and background processes and scheduled automations may outlive a stop. — [src/agents/timeout.ts:13](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/src/agents/timeout.ts#L13); [packages/agent-core/src/harness/env/kill-tree.ts:32-36](https://github.com/openclaw/openclaw/blob/4c0c0c0e3af9310aef3ca8c765a498ca47f9d1d7/packages/agent-core/src/harness/env/kill-tree.ts#L32-L36) (verified)
  - *To reach the next level:* Tight per-run ceilings, with stop cancelling pending calls and scheduled work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web_fetch/web_search/browser and inbound channel messages (src/agents/tool-catalog.ts:142) · [B] sensitive data/systems: host filesystem and inherited provider/channel keys (src/agents/bash-tools.exec-request-preparation.ts:511) · [C] state change / egress: message tool and host shell exec without approval (src/agents/tool-catalog.ts:358, src/infra/exec-approvals-config.ts:95) · Same default session? Yes

## Highest-impact improvements
1. Default host exec to tools.exec.mode 'ask' (allowlist + on-miss) instead of full/off. — C2 D L0→L3, +0.150 before caps (Playbook 5)
2. Default the sandbox to 'all' when Docker is available, failing closed otherwise. — C4 D L0→L3, +0.150 before caps (Playbook 3 step 1)
3. Require human approval for model-initiated plugin installs. — C7 D L0→L2, +0.100 before caps (Playbook 3)
4. Default tools.fs.workspaceOnly to true and deny writes to the OpenClaw state/config directory. — C3 C L1→L2, +0.075 before caps (Playbook 3)
5. Use the existing turn-taint signal to require approval for egress and exec after untrusted content. — C5 S L1→L3, +0.150 before caps (Playbook 1)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The repository is very large (~52k files); the review focused on the Gateway core (src/agents, src/infra, src/security, src/plugins, src/gateway, src/audit). Channel extensions, native apps (apps/), the Codex/ACP harnesses and node-host execution were not examined in depth.
- Scored the README-led local npm + onboarding install; the Docker Compose deployment (non-root, cap_drop NET_RAW/NET_ADMIN) and remote/team deployments were not scored separately.
- Whether shell output sent to the model is pattern-redacted was not traced end to end; C8 rated conservatively.
- No text aimed at AI reviewers was found in repository markdown.
