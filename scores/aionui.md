# Defense-in-Depth Score: AionUi

**Repo:** https://github.com/iofficeai/aionui · **Commit:** `6744099b279b991c17e31c243f0920477bd31cb6` (v2.2.2) · **Reviewed:** 2026-10-04
**What it is:** Electron desktop and WebUI 'cowork' app that wraps CLI coding agents (Claude Code, Gemini, Codex and others via ACP) and a built-in agent, with MCP, skills, scheduled tasks, teams and chat channels.
**Category:** AI Assistants
**Scored configuration:** Packaged desktop app on first launch with default settings (built-in browser MCP enabled, release-build Sentry DSN), scored on the code in this repository only.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication opt-in

## Score: 1.5 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C2 | Approval gates | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | Low |
| C3 | Tool & action scoping | L1 | L1 | L0 | L1 | 0.20 | — | **0.20** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | Low |
| C7 | Third-party extensions | L1 | L1 | L1 | L0 | 0.20 | C7-RCELOAD | **0.20** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L0 | L1 | 0.20 | — | **0.20** | Medium |
| C9 | Audit & traceability | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | Low |
| C10 | Limits & kill switch | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |


AionUi is the desktop front end and launcher. The agent runtime, including tool execution and enforcement of approvals, lives in the separate AionCore backend, which was not reviewed, so most controls cannot be credited here. What this repo does ship is risky: the backend and MCP servers inherit the user's full environment with no sandbox; a browser tool, on by default, runs an npm package fetched by npx inside a browser that keeps the user's sign-ins; chat output rendering is not locked down; and scheduled tasks always run in YOLO mode. The approval card does show the exact command, and the browser bridge is carefully limited to one webview.

## Critical gaps
- Agent backend and MCP processes run unsandboxed as the user with the full inherited environment, so executed code is host-equivalent and holds every credential in the environment. (ASI05, T11; C4) — [packages/web-host/src/backend-launcher.ts:230-236](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L230-L236); [packages/web-host/src/backend-launcher.ts:692](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L692); [packages/desktop/src/process/resources/builtinMcp/browserServer.ts:158-162](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/browserServer.ts#L158-L162)
- The default-on browser MCP runs `npx -y chrome-devtools-mcp@0.16.0` without consent or an integrity check and gives the downloaded code the full inherited environment. (ASI04, T17, LLM03; C7) — [packages/desktop/src/process/resources/builtinMcp/browserServerPort.ts:105](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/browserServerPort.ts#L105); [packages/desktop/src/process/resources/builtinMcp/browserServer.ts:108](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/browserServer.ts#L108); [packages/desktop/src/process/resources/builtinMcp/browserServer.ts:158-162](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/browserServer.ts#L158-L162)

## Criterion details

### C1 Identity & least privilege — 0.05 (high)

AionUi hands the agent backend the user's whole environment: the backend process is spawned with every parent environment variable (cloud keys, tokens, anything in the shell), and the built-in browser tool runs in an in-app browser whose sign-in cookies are shared across tabs and kept between sessions. Nothing in this repository narrows that authority, scopes credentials per tool, or checks authorization per request; whatever per-request checks exist live in the separate AionCore backend, which was not reviewed. A hijacked agent therefore acts with the user's local authority plus any websites they signed into inside the app.

- **S L0:** The backend and its agents inherit the full parent environment and the persistent in-app browser session; no scoped identity exists in this repo. — [packages/web-host/src/backend-launcher.ts:230-236](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L230-L236); [packages/web-host/src/backend-launcher.ts:692](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L692); [packages/desktop/src/process/utils/runBackendMigrations.ts:188-189](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/utils/runBackendMigrations.ts#L188-L189) (verified)
  - *To reach the next level:* No per-tool or per-capability credential scoping; the backend receives every inherited environment variable.
- **C L0:** No authorization layer exists in the host; the built-in browser MCP launcher also receives process.env whole. — [packages/web-host/src/backend-launcher.ts:230-236](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L230-L236); [packages/web-host/src/backend-launcher.ts:692](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L692); [packages/desktop/src/process/resources/builtinMcp/browserServer.ts:158-162](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/browserServer.ts#L158-L162) (verified)
  - *To reach the next level:* No tool path in this repo passes an authorization check before acting with the inherited credentials.
- **D L0:** The default install runs with the user's full ambient environment; nothing narrower is configured. — [packages/web-host/src/backend-launcher.ts:230-236](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L230-L236); [packages/web-host/src/backend-launcher.ts:692](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L692) (verified)
  - *To reach the next level:* No minimal default identity; least privilege would require the user to launch the app from a scrubbed environment.
- **B L1:** A hijack reaches the user's files, every inherited credential, and signed-in sites in the shared browser partition: write access across several systems. — [packages/web-host/src/backend-launcher.ts:230-236](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L230-L236); [packages/web-host/src/backend-launcher.ts:692](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L692); [packages/desktop/src/process/utils/runBackendMigrations.ts:188-189](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/utils/runBackendMigrations.ts#L188-L189) (verified)
  - *To reach the next level:* Authority is not limited to one system or one project.
- **Cap:** none

### C2 Approval gates — 0.33 (low)

When an agent asks for permission, AionUi's approval card shows the exact command, or the raw tool input as JSON, along with the option buttons the agent offered. That is a good way to show the request. But the decision about which actions need approval, and whether the answer is enforced, belongs to the AionCore backend and the wrapped agent CLIs, which this repo does not contain. Scheduled tasks are always created in the agent's YOLO (auto-approve) mode, so background runs skip approval entirely, and a new chat can start in whatever permission mode was used last.

- **S L2:** The approval card renders the raw command or full raw_input JSON, but the title is agent-written and risk tiers and enforcement live outside this repo. — [packages/desktop/src/renderer/pages/conversation/Messages/acp/MessageAcpPermission.tsx:67-82](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/renderer/pages/conversation/Messages/acp/MessageAcpPermission.tsx#L67-L82); [packages/desktop/src/renderer/pages/conversation/Messages/components/MessagePermission/PermissionRequestPanel.tsx:121-125](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/renderer/pages/conversation/Messages/components/MessagePermission/PermissionRequestPanel.tsx#L121-L125) (verified)
  - *To reach the next level:* Risk tiers and the execution-matches-approval guarantee are not visible in this repo; only the presentation layer is.
- **C L1:** Scheduled (cron) tasks always run in YOLO mode, so a background path bypasses the gate; whether every interactive tool is gated is decided in AionCore (not reviewed). — [packages/desktop/src/renderer/pages/cron/ScheduledTasksPage/CreateTaskDialog.tsx:363-369](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/renderer/pages/cron/ScheduledTasksPage/CreateTaskDialog.tsx#L363-L369); [packages/desktop/src/renderer/pages/cron/ScheduledTasksPage/resolveCronAgentConfig.ts:56](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/renderer/pages/cron/ScheduledTasksPage/resolveCronAgentConfig.ts#L56) (inferred)
  - *To reach the next level:* Background tasks bypass the gate and coverage of built-in and MCP tools cannot be shown from this repo.
- **D L1:** Cron jobs default to auto-approve with no per-task choice, and the per-assistant permission default can follow the last value used. — [packages/desktop/src/renderer/pages/cron/ScheduledTasksPage/CreateTaskDialog.tsx:363-369](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/renderer/pages/cron/ScheduledTasksPage/CreateTaskDialog.tsx#L363-L369); [packages/desktop/src/renderer/pages/cron/ScheduledTasksPage/resolveCronAgentConfig.ts:56](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/renderer/pages/cron/ScheduledTasksPage/resolveCronAgentConfig.ts#L56); [packages/desktop/src/renderer/pages/guid/utils/assistantDefaults.ts:33-38](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/renderer/pages/guid/utils/assistantDefaults.ts#L33-L38) (inferred)
  - *To reach the next level:* Auto-approve is the shipped default for scheduled tasks, and elevated modes are not session- or time-bounded.
- **B L1:** No checkpoint or rollback exists in this repo, and the default-on browser tool can click and type in signed-in websites, which can't be undone. — [packages/desktop/src/process/utils/runBackendMigrations.ts:186-192](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/utils/runBackendMigrations.ts#L186-L192); searched `rg -n -i 'checkpoint|rollback|undo'` in `packages/desktop/src/common/adapter/ipcBridge.ts` → 0 hits (The client API exposes no checkpoint, rollback or undo endpoint.) (inferred)
  - *To reach the next level:* No checkpoints or rollback for file or external actions are shown in this repo.
- **Cap:** none

### C3 Tool & action scoping — 0.20 (high)

The one tool this repo ships and turns on by default is the in-app browser MCP (chrome-devtools-mcp). Its connection is carefully limited: the bridge listens only on localhost, needs a random token, and attaches only to the side-panel webview, never the main window. Inside that page, though, the agent can navigate to any URL, click, type and run scripts in a browser that holds the user's sign-in cookies. Every other tool lives in AionCore and the wrapped CLIs and was not reviewed.

- **S L1:** The CDP bridge restricts which target can be driven (webview only, loopback, token), but commands inside that target pass through with no URL or argument validation. — [packages/desktop/src/process/resources/builtinMcp/cdpBridge.ts:36](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/cdpBridge.ts#L36); [packages/desktop/src/process/resources/builtinMcp/cdpBridge.ts:105-111](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/cdpBridge.ts#L105-L111) (verified)
  - *To reach the next level:* No URL/host allowlist or argument validation on browser actions; general navigation and script tools remain.
- **C L1:** The target restriction applies only to the built-in browser MCP; no shared validation layer exists for other MCP or agent tools in this repo. — [packages/desktop/src/process/resources/builtinMcp/cdpBridge.ts:105-111](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/cdpBridge.ts#L105-L111) (verified)
  - *To reach the next level:* No validation layer covers the other built-in or user-added tools.
- **D L0:** The browser MCP is registered enabled by default, alongside the agents' write and exec tools. — [packages/desktop/src/process/utils/runBackendMigrations.ts:186-192](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/utils/runBackendMigrations.ts#L186-L192) (verified)
  - *To reach the next level:* Write, exec and browser tools are all on by default; there is no read-only default tool set.
- **B L1:** A misused browser tool reaches any website with the user's persisted sign-in state; agent shell and file tools reach the whole machine. — [packages/desktop/src/process/utils/runBackendMigrations.ts:188-189](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/utils/runBackendMigrations.ts#L188-L189) (verified)
  - *To reach the next level:* Reach is not limited to a project or bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

AionUi contains no isolation layer of its own. The backend that runs agents, and the MCP servers it launches, start as ordinary processes running as the user, with the full inherited environment. Some wrapped CLIs have their own sandbox modes (the UI has labels such as 'yoloNoSandbox' and Codex 'sandboxMode' settings), but those belong to third-party agents and are not enforced here. If agent-run code goes wrong, it has host-level access, including every credential in the environment.

- **S L0:** No sandbox primitive exists in the host code; the backend and MCP servers are plain same-user subprocesses. — searched `rg -n -i 'seccomp|bwrap|bubblewrap|firejail|sandbox-exec|landlock|docker|gvisor|firecracker'` in `packages/web-host/src packages/desktop/src/process` → 0 hits (No isolation primitive anywhere in the host code that launches the backend and MCP processes.); [packages/web-host/src/backend-launcher.ts:230-236](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L230-L236); [packages/web-host/src/backend-launcher.ts:692](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L692) (verified)
  - *To reach the next level:* No OS-level or container isolation around agent-executed code.
- **C L0:** No execution path launched from this repo is sandboxed, including the npx-launched browser MCP. — [packages/desktop/src/process/resources/builtinMcp/browserServer.ts:158-162](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/browserServer.ts#L158-L162) (verified)
  - *To reach the next level:* No execution path goes through an isolation boundary.
- **D L0:** No isolation is on by default, because none exists in this repo. — searched `rg -n -i 'seccomp|bwrap|bubblewrap|firejail|sandbox-exec|landlock|docker|gvisor|firecracker'` in `packages/web-host/src packages/desktop/src/process` → 0 hits (No isolation primitive anywhere in the host code that launches the backend and MCP processes.) (verified)
  - *To reach the next level:* Isolation is neither present nor on by default.
- **B L0:** Code runs host-equivalent with the full inherited environment, so credentials sit in the execution environment. — [packages/web-host/src/backend-launcher.ts:230-236](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L230-L236); [packages/web-host/src/backend-launcher.ts:692](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L692); [packages/desktop/src/process/resources/builtinMcp/browserServer.ts:158-162](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/browserServer.ts#L158-L162) (verified)
  - *To reach the next level:* Credentials and the home directory are reachable from executed code.
- **Cap:** none

### C5 Untrusted input blast radius — 0.05 (high)

Out of the box the agent has a browser tool that reads arbitrary web pages, which is untrusted content, inside a browser that keeps the user's sign-ins. Nothing in this repo tracks untrusted content or restricts what the agent may do after reading it. The chat and preview renderers are not locked down. Whether irreversible actions are approved depends on the backend and the agent CLIs, which were not reviewed.

- **S L0:** No provenance tagging, taint tracking or Rule-of-Two enforcement exists in the host; nothing limits a hijacked agent here. — [packages/desktop/src/process/utils/runBackendMigrations.ts:186-192](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/utils/runBackendMigrations.ts#L186-L192) (verified)
  - *To reach the next level:* No structural limit applies once untrusted content has been read.
- **C L0:** Web pages, tool results and MCP outputs are not distinguished from user input anywhere in this repo. — searched `rg -n -i content-security-policy` in `packages/desktop/src packages/web-host/src` → 0 hits (No Content-Security-Policy is set by the host or renderer.) (verified)
  - *To reach the next level:* No untrusted source is identified or handled differently.
- **D L0:** No control exists to be on by default; the browser tool that ingests untrusted pages is itself on by default. — [packages/desktop/src/process/utils/runBackendMigrations.ts:186-192](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/utils/runBackendMigrations.ts#L186-L192) (verified)
  - *To reach the next level:* No untrusted-input control ships on.
- **B L1:** Output rendering is not locked down, which is relevant to exfiltration; irreversible actions depend on agent-side approval that this repo cannot show. — searched `rg -n -i content-security-policy` in `packages/desktop/src packages/web-host/src` → 0 hits (No Content-Security-Policy is set by the host or renderer.) (verified)
  - *To reach the next level:* Exfiltration is not gated by approval.
- **Cap:** none
- **Notes:** The chat and preview renderers are not locked down.

### C6 Memory, context & configuration integrity — 0.10 (low)

Conversations, assistant rules, imported skills and scheduled-task skills are all stored by the AionCore backend, and the client offers endpoints to write assistant rules, import skills and turn on a skills market. This repo has no validation, provenance tagging or review step for anything persisted and later loaded into agent context. Workspace instruction files are loaded by the wrapped CLIs themselves. Persisted rules and skills carry over into the user's later sessions and can steer tool use.

- **S L0:** Rule and skill writes go straight to backend endpoints, with no validation or approval visible in the host. — [packages/desktop/src/common/adapter/ipcBridge.ts:903-905](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/common/adapter/ipcBridge.ts#L903-L905); [packages/desktop/src/common/adapter/ipcBridge.ts:1000](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/common/adapter/ipcBridge.ts#L1000) (inferred)
  - *To reach the next level:* No gating, validation or provenance on persisted rules, skills or memory in this repo.
- **C L0:** No persistence path is controlled from this repo. — [packages/desktop/src/common/adapter/ipcBridge.ts:942](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/common/adapter/ipcBridge.ts#L942) (inferred)
  - *To reach the next level:* No memory or auto-loaded config path is controlled.
- **D L1:** This is a single-user desktop app (WebUI has one admin user), so cross-user sharing is limited by deployment shape rather than enforced namespaces. — [packages/web-cli/src/ensureAdminPassword.ts:5-9](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-cli/src/ensureAdminPassword.ts#L5-L9) (inferred)
  - *To reach the next level:* No per-user or per-session namespace enforcement can be shown in this repo.
- **B L1:** Poisoned rules or skills persist across the user's sessions and are loaded into agents that can use tools. — [packages/desktop/src/common/adapter/ipcBridge.ts:903-905](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/common/adapter/ipcBridge.ts#L903-L905) (inferred)
  - *To reach the next level:* No session scoping or human review before persisted context is reused.
- **Cap:** none

### C7 Third-party extensions — 0.20 (high)

On first use, the default-on browser tool runs `npx -y chrome-devtools-mcp@0.16.0`. That downloads and runs an npm package without asking the user. The version is pinned, but there is no hash check and its dependencies are not locked. A second built-in entry (off by default) uses `@latest`, and MCP servers the user adds are not pinned or verified by this repo. The launcher passes its whole environment to the downloaded server, so a compromised package would get every credential the app inherited.

- **S L1:** The built-in browser MCP pins the top-level version but has no integrity check; the default-off chrome-devtools entry uses @latest. — [packages/desktop/src/process/resources/builtinMcp/browserServerPort.ts:105](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/browserServerPort.ts#L105); [packages/desktop/src/process/resources/builtinMcp/browserServer.ts:108](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/browserServer.ts#L108); [packages/desktop/src/process/utils/runBackendMigrations.ts:204-214](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/utils/runBackendMigrations.ts#L204-L214) (verified)
  - *To reach the next level:* No hash or signature verification and no locked transitive dependencies for npx-fetched servers.
- **C L1:** Only the built-in browser server is version-pinned; user-added MCP servers, skills and extensions get no verification in this repo. — [packages/desktop/src/process/resources/builtinMcp/browserServerPort.ts:105](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/browserServerPort.ts#L105); [packages/desktop/src/process/resources/builtinMcp/browserServer.ts:108](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/browserServer.ts#L108) (verified)
  - *To reach the next level:* Pinning covers one extension; other types are unverified.
- **D L1:** The browser MCP is enabled by default and fetched by npx -y on first agent use without showing the user what will run. — [packages/desktop/src/process/utils/runBackendMigrations.ts:186-192](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/utils/runBackendMigrations.ts#L186-L192); [packages/desktop/src/process/resources/builtinMcp/browserServerPort.ts:105](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/browserServerPort.ts#L105); [packages/desktop/src/process/resources/builtinMcp/browserServer.ts:108](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/browserServer.ts#L108) (verified)
  - *To reach the next level:* Third-party code is enabled by default without consent.
- **B L0:** The npx-launched server inherits process.env whole, as the same user with the full environment. — [packages/desktop/src/process/resources/builtinMcp/browserServer.ts:158-162](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/resources/builtinMcp/browserServer.ts#L158-L162) (verified)
  - *To reach the next level:* No scrubbed environment or per-extension sandbox.
- **Cap:** C7-RCELOAD — By default the app has npx fetch and execute a remote npm package (chrome-devtools-mcp, plus its unlocked dependency tree) on first browser-tool use without user consent.

### C8 Secrets & sensitive-data protection — 0.20 (medium)

Release builds have a Sentry key built in, and once a day, starting 30 seconds after launch, the app uploads the last day's frontend and backend log files to Sentry as a gzipped attachment. There is no opt-in, and no redaction applies to those files. Redaction (tokens, emails, home-directory names) exists only for one path: error summaries attached to telemetry. The backend and MCP servers also receive the full environment, so long-lived keys in the shell reach every agent process.

- **S L1:** Masking exists for error-summary telemetry only; uploaded log files and subprocess environments are unfiltered. — [packages/desktop/src/renderer/pages/conversation/platforms/acp/errorDiagnostics.ts:53](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/renderer/pages/conversation/platforms/acp/errorDiagnostics.ts#L53); [packages/web-host/src/backend-launcher.ts:230-236](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L230-L236); [packages/web-host/src/backend-launcher.ts:692](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L692) (verified)
  - *To reach the next level:* No redaction on log uploads or model-bound messages; no secret-store usage shown in this repo.
- **C L1:** Only the error-summary telemetry path is redacted. — [packages/desktop/src/renderer/pages/conversation/platforms/acp/errorDiagnostics.ts:53](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/renderer/pages/conversation/platforms/acp/errorDiagnostics.ts#L53); [packages/desktop/src/sentry.ts:490-534](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/sentry.ts#L490-L534) (verified)
  - *To reach the next level:* Logs, uploaded log files and subprocess environments are unprotected.
- **D L0:** Crash reporting is on by default in release builds and uploads whole log files (including renderer-forwarded log data) to a third party; whether they contain prompts is inferred from the logging paths, not observed. — [.github/workflows/_build-reusable.yml:435](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/.github/workflows/_build-reusable.yml#L435); [packages/desktop/src/sentry.ts:118](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/sentry.ts#L118); [packages/desktop/src/index.ts:507](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/index.ts#L507); [packages/desktop/src/process/bridge/applicationBridge.ts:162-173](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/process/bridge/applicationBridge.ts#L162-L173) (inferred)
  - *To reach the next level:* Log upload is not opt-in and not content-free.
- **B L1:** Long-lived provider API keys and any inherited environment secrets reach every agent subprocess. — [packages/web-host/src/backend-launcher.ts:230-236](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L230-L236); [packages/web-host/src/backend-launcher.ts:692](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L692) (verified)
  - *To reach the next level:* Keys are not scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.38 (low)

Agent tool calls and permission requests are stored as typed conversation messages ('acp_tool_call', 'acp_permission') and fetched from the backend's conversation API, so a local transcript exists. Writing and storing that record happens in AionCore, which was not reviewed, so its completeness, its attribution of who approved what, and its durability cannot be shown here. The record lives in the app's own data directory, which the agent's own unsandboxed tools can reach.

- **S L2:** Typed tool-call and permission messages are persisted per conversation and served by the backend. — [packages/desktop/src/common/chat/chatLib.ts:301](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/common/chat/chatLib.ts#L301); [packages/desktop/src/common/chat/chatLib.ts:329](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/common/chat/chatLib.ts#L329); [packages/desktop/src/common/adapter/ipcBridge.ts:1372](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/common/adapter/ipcBridge.ts#L1372) (inferred)
  - *To reach the next level:* No actor attribution or approver identity is visible on records.
- **C L1:** The main conversation path is recorded; coverage of cron, team sub-agents and MCP calls cannot be shown from this repo. — [packages/desktop/src/common/adapter/ipcBridge.ts:1372](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/common/adapter/ipcBridge.ts#L1372) (inferred)
  - *To reach the next level:* No evidence that all tool calls, approvals and denials are recorded.
- **D L2:** Records sit in the app data directory, outside the workspace, but unsandboxed agent tools running as the same user can edit them. — [packages/web-host/src/backend-launcher.ts:201-202](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/backend-launcher.ts#L201-L202) (inferred)
  - *To reach the next level:* The record is not written by a component the model can't control.
- **B L1:** Durability and flush behaviour are backend concerns that can't be shown; log upload and process cleanup are explicitly best-effort. — [packages/web-host/src/agent-process-registry.ts:38-44](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/agent-process-registry.ts#L38-L44) (inferred)
  - *To reach the next level:* No evidence of per-action durable writes.
- **Cap:** none

### C10 Limits & kill switch — 0.00 (high)

This repo sets no step, time or cost limit on agent runs. The only hit is an optional 'maxTurns' field with no default. There is a working stop: the UI can cancel a conversation or a team run, and on shutdown the app kills registered agent process groups (SIGTERM, then SIGKILL). Scheduled tasks keep firing on their own schedule, and runaway work is not bounded except by the provider account.

- **S L0:** A halt exists (cancel endpoint, process-group kill at shutdown) but there is no step, time or cost limit. — searched `rg -n -i 'max_?steps|max_?iterations|max_?turns|cost_?limit|spend_?limit|token_?budget'` in `packages/web-host/src packages/desktop/src/process packages/desktop/src/common` → 1 hits (The single hit is an optional per-conversation maxTurns type field (storage.ts:477) with no default or enforcement in this repo.); [packages/desktop/src/common/adapter/ipcBridge.ts:361](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/desktop/src/common/adapter/ipcBridge.ts#L361) (verified)
  - *To reach the next level:* No iteration, wall-clock or cost cap enforced in code.
- **C L0:** No limit applies to any path. — searched `rg -n -i 'max_?steps|max_?iterations|max_?turns|cost_?limit|spend_?limit|token_?budget'` in `packages/web-host/src packages/desktop/src/process packages/desktop/src/common` → 1 hits (The single hit is an optional per-conversation maxTurns type field (storage.ts:477) with no default or enforcement in this repo.) (verified)
  - *To reach the next level:* No limits exist to cover the loop, tools or sub-agents.
- **D L0:** Unlimited by default. — searched `rg -n -i 'max_?steps|max_?iterations|max_?turns|cost_?limit|spend_?limit|token_?budget'` in `packages/web-host/src packages/desktop/src/process packages/desktop/src/common` → 1 hits (The single hit is an optional per-conversation maxTurns type field (storage.ts:477) with no default or enforcement in this repo.) (verified)
  - *To reach the next level:* No default limits.
- **B L0:** No ceiling; stop only kills registered processes, and scheduled tasks continue firing. — [packages/web-host/src/agent-process-registry.ts:53](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/agent-process-registry.ts#L53); [packages/web-host/src/agent-process-registry.ts:169](https://github.com/iofficeai/aionui/blob/6744099b279b991c17e31c243f0920477bd31cb6/packages/web-host/src/agent-process-registry.ts#L169) (verified)
  - *To reach the next level:* No per-run ceiling on time or spend.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Default-on in-app browser MCP reads arbitrary web pages (packages/desktop/src/process/utils/runBackendMigrations.ts:192) · [B] sensitive data/systems: Persistent shared sign-in partition and full inherited environment (runBackendMigrations.ts:189; packages/web-host/src/backend-launcher.ts:233) · [C] state change / egress: Browser click/type in signed-in sites · Same default session? Yes

## Highest-impact improvements
1. Harden chat output rendering and set a restrictive CSP. — C5 B L1→L2, +0.050 before caps (Playbook 1)
2. Make the startup log upload to Sentry opt-in and redact log files before upload. — C8 D L0→L2, +0.100 before caps (Playbook 4)
3. Launch the browser MCP and backend with an allowlisted environment instead of process.env. — C7 B L0→L2, +0.100 before caps (Playbook 3)
4. Ship the built-in browser MCP disabled (or vendored and hash-verified instead of npx -y) and show what will run before enabling it. — C3 D L0→L2, +0.100 before caps (Playbook 3)
5. Add per-run wall-clock and turn caps enforced by the host, including for scheduled tasks. — C10 S L0→L2, +0.150 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The agent runtime (tool execution, approval enforcement, MCP client, cron executor, channels, extensions, persistence) lives in the separate AionCore repository (docs/contributing/development.md:17-18), which was not reviewed; controls there are not credited, and several ratings are INFERRED for that reason.
- Wrapped third-party agent CLIs (Claude Code, Gemini CLI, Codex, etc.) bring their own approval and sandbox mechanisms; they are out of scope.
- The WebUI/remote mode (--remote binds 0.0.0.0) and chat channels (Telegram, Slack, Lark) were examined only at the client layer.
- No text aimed at AI reviewers was found in the repository.
