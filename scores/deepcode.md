# Defense-in-depth score: DeepCode

**Repo:** https://github.com/HKUDS/DeepCode · **Commit:** `84c37f79c726bd12e74e39541adceb8b5e47b0d5` (v2.3.0) · **Reviewed:** 2026-10-05
**What it is:** Local agentic coding assistant with TUI, Desktop and Web clients over a shared background service, plus the Paper2Code multi-agent workflow.
**Category:** Coding
**Scored configuration:** The `deepcode` TUI in a trusted project with the default Ask access preset, on Linux or macOS with the bubblewrap or Seatbelt sandbox backend available, no extra environment variables.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents yes · external communication yes

## Score: 3.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L1 | L1 | 0.25 | C1-SELFESC | **0.25** | High |
| C2 | Approval gates | L2 | L2 | L1 | L2 | 0.45 | G2 | **0.25** | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | none | **0.45** | High |
| C4 | Code-execution isolation | L2 | L1 | L1 | L0 | 0.28 | G2 | **0.25** | High |
| C5 | Untrusted input blast radius | L2 | L2 | L1 | L1 | 0.40 | none | **0.40** | High |
| C6 | Memory, context & configuration integrity | L2 | L2 | L2 | L1 | 0.45 | none | **0.45** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L2 | 0.28 | none | **0.28** | High |
| C8 | Secrets & sensitive-data protection | L2 | L1 | L2 | L0 | 0.33 | none | **0.33** | High |
| C9 | Audit & traceability | L2 | L3 | L2 | L3 | 0.62 | none | **0.62** | High |
| C10 | Limits & kill switch | L2 | L2 | L0 | L0 | 0.30 | G1 | **0.30** | High |


DeepCode asks before it runs commands or changes files by default, and runs shell commands in a bubblewrap or Seatbelt sandbox that keeps writes inside the workspace. That sandbox leaves the network open, the home directory readable and the full environment (provider keys included) visible, and commands run without it when no backend is installed. Reading files outside a short credential denylist and fetching any public URL need no approval, so a hijacked session can read local data and send it out without a human. Approval and sandbox settings can be loosened without an explicit operator decision, and there is no default cap on steps or spend.

## Critical gaps
- The agent's own tools are not prevented from changing settings that govern its permissions on later turns. (ASI03, T3; C1). Evidence: [core/harness/sandbox.py:254-256](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/sandbox.py#L254-L256)
- Approval settings can be loosened without an explicit operator decision in the default configuration. (ASI09, ASI02, T10; C2). Evidence: [core/config.py:396-403](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/config.py#L396-L403)
- Sandbox settings can be loosened without an explicit operator decision; commands run unsandboxed when no backend is installed. (ASI05, T11; C4). Evidence: [core/harness/sandbox.py:336](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/sandbox.py#L336)
- The command sandbox can read the home directory and receives provider keys in its environment, with network open. (ASI05, T11, LLM05; C4). Evidence: [core/harness/sandbox.py:309-318](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/sandbox.py#L309-L318); [core/harness/env_sanitize.py:75-85](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/env_sanitize.py#L75-L85)

## Criterion details

### C1 Identity & least privilege: 0.25 (high confidence)

DeepCode runs as the local user and narrows that authority only a little. File tools refuse a short list of credential locations, MCP servers and external sub-agent CLIs get only the environment they are configured with, and a project configuration cannot redirect the user's provider keys. The agent's own shell, code mode and hooks still receive the full parent environment, including provider keys, unless the operator opts into scrubbing, and the agent's own tools are not prevented from changing settings that govern later turns.

- **S L1:** Ambient user authority with modest narrowing: credential scrubbing exists but the shell gets the full environment by default; MCP stdio servers get an explicit environment. Evidence: [core/harness/env_sanitize.py:75-85](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/env_sanitize.py#L75-L85); [core/mcp/connection.py:217-222](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/mcp/connection.py#L217-L222) (verified)
  - *To reach the next level:* No scoped or per-tool credentials and no default environment scrubbing for the agent's own shell.
- **C L1:** The shell, hooks and code mode inherit the full ambient environment; only MCP servers and external CLIs are narrowed. Evidence: [core/harness/tools/shell.py:203-211](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/tools/shell.py#L203-L211); [core/harness/hooks/execution.py:70-72](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/hooks/execution.py#L70-L72) (verified)
  - *To reach the next level:* Every tool path, including the shell and hooks, should run with a narrowed environment and pass the same authorization layer.
- **D L1:** Environment scrubbing for the shell is opt-in through an environment variable; the default hands children everything. Evidence: [core/harness/env_sanitize.py:56-58](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/env_sanitize.py#L56-L58) (verified)
  - *To reach the next level:* Least-privilege environment should be the default, with widening an explicit operator choice.
- **B L1:** Commands see the user's whole home directory read-only plus provider keys and any CLI tokens in the environment; the approval gate on the shell is the surviving layer. Evidence: [core/harness/sandbox.py:309-318](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/sandbox.py#L309-L318) (verified)
  - *To reach the next level:* Reach should be limited to one project and mostly read, not the user's accounts across services.
- **Cap:** C1-SELFESC: The agent's own tools are not prevented from changing settings that govern its permissions on later turns.
- **Notes:** The cap's precondition is a shell command the user approves; the approval prompt is the surviving control.

### C2 Approval gates: 0.25 (high confidence)

New sessions default to an Ask preset: read-only tools run freely and every other built-in, MCP and sub-agent tool call stops for a durable approval that the user can grant once, for the session, or deny. Denials and approver errors fail closed, and the local service only accepts approvals from authenticated clients. In the TUI the prompt shows the tool name and reason with the command clipped to the terminal width and no file diff, a session grant covers every later call of that tool, MCP tools that call themselves read-only and the URL fetcher are never gated, and approval settings can be loosened without an explicit operator decision.

- **S L2:** Per-call approval with full arguments stored, but the TUI approval prompt shows only the tool name and reason next to a clipped command card. Evidence: [cli/tui/app.py:853-862](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/cli/tui/app.py#L853-L862); [core/application/approval_service.py:98-104](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/application/approval_service.py#L98-L104) (verified)
  - *To reach the next level:* The approver should see the exact full command, arguments and file diff, with argument-level policy on parsed arguments.
- **C L2:** Built-in mutating tools, MCP tools and sub-agents share one engine, but web fetch is auto-allowed and MCP tools are gated by their own read-only hint. Evidence: [core/harness/permissions.py:103-120](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/permissions.py#L103-L120); [core/agent_setup.py:471-479](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/agent_setup.py#L471-L479) (verified)
  - *To reach the next level:* Auto-approved tools should be a verified read-only allowlist with no outbound channel, not server-declared hints.
- **D L1:** Ask is the default preset, but approval settings can be loosened without an explicit operator decision in the default configuration. Evidence: [core/application/execution_security_policy.py:137-141](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/application/execution_security_policy.py#L137-L141); [core/config.py:396-403](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/config.py#L396-L403) (verified)
  - *To reach the next level:* Any loosening of approval settings should require an explicit, confirmed operator action.
- **B L2:** Sandboxed writes stay in the workspace, where git usually makes them reversible, but approved commands can push, delete in the workspace or call the network irreversibly; the shadow-snapshot helper is not wired in. Evidence: searched `rg -n 'harness.snapshot|Snapshotter'` in `core cli app_server` → 1 hits (the only hit is the class definition; nothing imports or uses the snapshotter); [core/harness/sandbox.py:410-413](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/sandbox.py#L410-L413) (verified)
  - *To reach the next level:* Checkpoints or rollback for workspace changes and previews for external actions.
- **Cap:** G2: Approval settings can be loosened without an explicit operator decision in the default configuration.

### C3 Tool & action scoping: 0.45 (high confidence)

Write, edit and patch tools resolve paths and refuse anything outside the workspace, and the URL fetcher only reaches public addresses on ports 80 and 443 and rechecks every redirect. The shell tool is a raw command string, the read tool accepts any path on the machine, and MCP tools get no shared argument validation. The default tool set includes write, shell and network tools.

- **S L2:** Resolved-path containment for writes and a strong SSRF guard for fetch, but the shell takes an arbitrary command and reads are unbounded. Evidence: [core/harness/tools/files.py:34-39](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/tools/files.py#L34-L39); [core/harness/tools/shell.py:197-201](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/tools/shell.py#L197-L201) (verified)
  - *To reach the next level:* Narrow tools in place of the general shell, and containment for reads.
- **C L2:** Most built-in tools validate their inputs; MCP tools only pass through the permission check. Evidence: [core/agent_setup.py:471-479](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/agent_setup.py#L471-L479) (verified)
  - *To reach the next level:* A shared validation layer that extension tools also go through.
- **D L2:** Access presets select a read-only group, but the default Ask set registers write, shell and web fetch. Evidence: [core/harness/tools/__init__.py:103-123](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/tools/__init__.py#L103-L123) (verified)
  - *To reach the next level:* A read-only tool set by default with write and exec enabled explicitly.
- **B L1:** Writes are fenced to the workspace, but reads reach the whole machine and the shell and fetch tools reach the network. Evidence: [core/harness/tools/files.py:28-31](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/tools/files.py#L28-L31); [core/harness/sandbox.py:383](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/sandbox.py#L383) (verified)
  - *To reach the next level:* Reads and network reach should be scoped to the project as well as writes.
- **Cap:** none

### C4 Code-execution isolation: 0.25 (high confidence)

Shell commands and code-mode programs run under bubblewrap on Linux or a deny-by-default Seatbelt profile on macOS, with writes limited to the workspace and temporary directories. Network stays open, the whole filesystem is readable and the full environment is passed in. Hooks and MCP stdio servers run directly on the host, commands run unsandboxed when no backend is installed, and sandbox settings can be loosened without an explicit operator decision.

- **S L2:** OS-level mount-namespace or Seatbelt separation with a workspace write fence, but network is allowed by default and there is no seccomp or capability hardening. Evidence: [core/harness/sandbox.py:309-318](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/sandbox.py#L309-L318); [core/harness/sandbox.py:383](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/sandbox.py#L383) (verified)
  - *To reach the next level:* Deny network by default and add seccomp or equivalent hardening.
- **C L1:** The shell and code mode are wrapped; hook commands and MCP stdio servers start on the host. Evidence: searched `rg -n 'create_subprocess_exec|build_exec_command'` in `core/harness/hooks` → 1 hits (hooks spawn directly; no sandbox wrapper is applied); [core/mcp/connection.py:224-227](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/mcp/connection.py#L224-L227) (verified)
  - *To reach the next level:* Hooks, MCP stdio servers and other spawned processes should go through the sandbox too.
- **D L1:** On by default where a backend exists, falls back to running the bare command when none is installed, and sandbox settings can be loosened without an explicit operator decision. Evidence: [core/harness/sandbox.py:336](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/sandbox.py#L336); [core/config.py:396-403](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/config.py#L396-L403) (verified)
  - *To reach the next level:* Fail closed without a backend and require an explicit, confirmed operator action to loosen sandbox settings.
- **B L0:** Inside the sandbox the home directory, including credential stores, is readable and provider keys are in the environment, with open network. Evidence: [core/harness/sandbox.py:309-318](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/sandbox.py#L309-L318); [core/harness/tools/shell.py:203-211](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/tools/shell.py#L203-L211) (verified)
  - *To reach the next level:* No secrets in the sandbox environment, home directory not mounted, egress off or allowlisted.
- **Cap:** G2: Sandbox settings can be loosened without an explicit operator decision; commands run unsandboxed when no backend is installed.

### C5 Untrusted input blast radius: 0.40 (high confidence)

Fetched pages and memory notes are wrapped in labels that mark them as untrusted, and every write, shell or unknown tool waits for approval whatever the agent has read. Nothing tracks what the session has read: reading local files outside the credential denylist and fetching any public URL stay unattended, and the UI shows fetched URLs without their query string. A hijacked session can therefore read local data and send it out without a human, while irreversible changes still need approval.

- **S L2:** Writes and command execution are always gated and fetched content is labelled untrusted, but outbound fetches are not gated. Evidence: [core/harness/tools/web.py:198-206](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/tools/web.py#L198-L206); [core/harness/permissions.py:376-382](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/permissions.py#L376-L382) (verified)
  - *To reach the next level:* Once untrusted content is read, every outbound and state-changing tool should require approval or be disabled.
- **C L2:** Web pages and memory are labelled; workspace files, MCP results and sub-agent messages enter context without provenance. Evidence: [core/harness/memory.py:431-436](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/memory.py#L431-L436) (verified)
  - *To reach the next level:* Tool results, MCP results and sub-agent messages should all carry provenance the runtime acts on.
- **D L1:** The approval-based limit is on by default but rests on approval settings that can be loosened without an explicit operator decision. Evidence: [core/application/execution_security_policy.py:137-141](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/application/execution_security_policy.py#L137-L141) (verified)
  - *To reach the next level:* Removing the limit should require an explicit, confirmed operator action.
- **B L1:** Unattended: reading files outside the denylist and fetching arbitrary URLs whose query string the UI hides; irreversible changes need approval. Evidence: [core/harness/permissions.py:103-120](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/permissions.py#L103-L120); [core/harness/tools/web.py:114-115](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/tools/web.py#L114-L115) (verified)
  - *To reach the next level:* Outbound requests should need approval once untrusted content is in the session.
- **Cap:** none

### C6 Memory, context & configuration integrity: 0.45 (high confidence)

The memory tool needs approval for every action in the default preset, and the MEMORY.md index is injected inside an untrusted-data boundary that escapes forged tags. Instruction files (AGENTS.md, DEEPCODE.md, CLAUDE.md) from the repository load silently as standing guidance, and project MCP servers, hooks, skills and settings apply once the folder is trusted through a folder-trust prompt. Memory has no expiry or versioning, and poisoned project files persist into every later session in that folder.

- **S L2:** Memory writes are approval-gated and presented as data; repository instruction files load silently, and a folder-trust prompt exists for project configuration. Evidence: [core/harness/memory.py:463-476](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/memory.py#L463-L476); [core/mcp/resolver.py:70-74](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/mcp/resolver.py#L70-L74) (verified)
  - *To reach the next level:* Memory expiry, and security-relevant project configuration confirmed item by item.
- **C L2:** The memory store is controlled; instruction files and project hook files are not. Evidence: [core/harness/hooks/discovery.py:17-24](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/hooks/discovery.py#L17-L24) (verified)
  - *To reach the next level:* All auto-loaded files and settings should be controlled, not just memory.
- **D L2:** Memory is namespaced per workspace under .deepcode/memory and the memory tool refuses names outside it. Evidence: [core/harness/memory.py:48-50](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/memory.py#L48-L50) (verified)
  - *To reach the next level:* The agent's approved shell can still write the workspace's own memory and instruction files.
- **B L1:** Poisoned instruction or memory files persist across the user's sessions in that project and can steer tool use. Evidence: [core/harness/memory.py:463-476](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/memory.py#L463-L476) (verified)
  - *To reach the next level:* Persistent content should only influence text or gated actions, or be easy to review and roll back.
- **Cap:** none

### C7 Third-party extensions: 0.28 (high confidence)

No third-party extension is enabled by default: bundled MCP presets are copied in disabled, and project MCP servers load only after the folder is trusted. When enabled, several presets launch unpinned `@latest` packages through npx or uvx, and nothing pins, hashes or re-approves an extension when its code or tool definitions change. MCP stdio servers run on the host with only the environment their configuration names.

- **S L1:** User-chosen extensions, launched from unpinned package references. Evidence: [core/mcp/presets.json:30-31](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/mcp/presets.json#L30-L31) (verified)
  - *To reach the next level:* Pinned versions for presets and user-added servers.
- **C L0:** No extension type is verified; the only hash or signature code in the MCP and plugin modules is filesystem change detection. Evidence: searched `rg -n -i 'checksum|signature|integrity'` in `core/mcp core/plugins` → 11 hits (all hits are stat-based change detection for plugin skill reloading, not integrity verification) (verified)
  - *To reach the next level:* Integrity checks for at least one extension type.
- **D L2:** Presets are added disabled and project servers wait for a folder-trust prompt. Evidence: [core/mcp/presets.py:76-79](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/mcp/presets.py#L76-L79) (verified)
  - *To reach the next level:* Adding an extension from any scope should show the exact package, command and permissions, and the workspace should not be able to add one.
- **B L2:** MCP stdio servers are separate processes started without the parent environment. Evidence: [core/mcp/connection.py:217-222](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/mcp/connection.py#L217-L222) (verified)
  - *To reach the next level:* Per-extension sandboxing and scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection: 0.33 (high confidence)

Saved provider keys live in a 0600 file in ~/.deepcode (keychain reads are opt-in), the UI never reads them back, configuration views mask credential fields, and provider error messages strip echoed keys. No analytics or crash-reporting SDK is present. The agent's shell and hooks receive every provider key in the environment by default, the LLM log keeps request and response previews, and the MCP log can hold credential arguments, which the project itself notes.

- **S L2:** Owner-only credential storage and redaction of keys in provider errors and config views. Evidence: [core/private_storage.py:23-24](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/private_storage.py#L23-L24); [core/providers/base.py:253-275](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/providers/base.py#L253-L275) (verified)
  - *To reach the next level:* OS keychain by default and redaction before logs and model-bound messages on all major paths.
- **C L1:** Error messages and extension subprocess environments are protected; logs, transcripts and the shell environment are not. Evidence: [core/observability/bus.py:28-30](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/observability/bus.py#L28-L30) (verified)
  - *To reach the next level:* Redaction in logs and transcripts as well as error paths.
- **D L2:** No telemetry; the local LLM log with prompt previews is on by default. Evidence: searched `rg -n -i 'sentry_sdk|posthog|mixpanel|amplitude'` in `core app_server cli` → 0 hits (no analytics or crash-reporting SDK); [core/config.py:514-518](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/config.py#L514-L518) (verified)
  - *To reach the next level:* Redaction always on for local logs.
- **B L0:** Long-lived provider keys in the environment reach every shell command and hook by default. Evidence: [core/harness/env_sanitize.py:75-85](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/env_sanitize.py#L75-L85) (verified)
  - *To reach the next level:* Keys kept out of child processes so exposed command output reveals nothing long-lived.
- **Cap:** none

### C9 Audit & traceability: 0.62 (high confidence)

Every tool call is stored as a structured item with its arguments and status in a local SQLite database under ~/.deepcode, outside the workspace, along with approval requests and decisions, and sub-agent transcripts are recorded. Records are written per action in transactions and the session can be replayed. There is no separate actor attribution, no tamper evidence for tool records (an optional hash chain covers only LLM calls) and no standard export.

- **S L2:** Structured per-call records with arguments, status and timestamps, including approvals. Evidence: [core/domain/item.py:20-37](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/domain/item.py#L20-L37); [core/application/approval_service.py:98-104](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/application/approval_service.py#L98-L104) (verified)
  - *To reach the next level:* Explicit actor, approver and delegation-chain attribution with correlation across sub-agents.
- **C L3:** Built-in, MCP and sub-agent calls flow through the same projection, and approvals and denials are recorded. Evidence: [core/agent_setup.py:386-399](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/agent_setup.py#L386-L399); [core/application/approval_service.py:129-160](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/application/approval_service.py#L129-L160) (verified)
  - *To reach the next level:* Configuration changes, memory writes and credential use are not recorded as audit events of their own.
- **D L2:** On by default and stored outside the workspace, but in a user-writable file that unsandboxed paths could alter. Evidence: [core/persistence/database.py:32-33](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/persistence/database.py#L32-L33) (verified)
  - *To reach the next level:* Written by a component the model cannot control on every platform.
- **B L3:** Records are committed per action in transactions and the session can be replayed. Evidence: [core/application/approval_service.py:80-90](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/application/approval_service.py#L80-L90) (verified)
  - *To reach the next level:* High-risk actions should not run unless their record is written.
- **Cap:** none

### C10 Limits & kill switch: 0.30 (high confidence)

Ordinary turns have no step limit by default; an iteration cap and Goal token budgets exist but are opt-in. Shell commands time out (120 seconds by default, but the model can choose a longer value), model requests have timeouts, sub-agents are limited to one level and five at once, and stopping a turn kills whole process groups. Repeated identical calls only trigger reminders, and there is no spend cap.

- **S L2:** An iteration cap, per-command timeouts and process-group kill on interrupt exist; the repeat guard only sends reminders. Evidence: [core/agent_runtime/runner.py:275-298](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/agent_runtime/runner.py#L275-L298); [core/agent_runtime/processes.py:34-42](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/agent_runtime/processes.py#L34-L42) (verified)
  - *To reach the next level:* Enforced token or cost caps and a repeated-action breaker.
- **C L2:** The loop and tool timeouts are bounded and sub-agent concurrency is capped, but there is no shared budget for sub-agents. Evidence: [core/harness/agents/control.py:44](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/agents/control.py#L44) (verified)
  - *To reach the next level:* Sub-agents and background work should count against the parent's budget.
- **D L0:** Unlimited by default: the iteration cap defaults to none and normal turns are unbounded. Evidence: [core/agent_runtime/runner.py:278-279](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/agent_runtime/runner.py#L278-L279); [cli/exec_cli.py:260](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/cli/exec_cli.py#L260) (verified)
  - *To reach the next level:* Sensible default step and spend limits.
- **B L0:** No ceiling on steps or spend, and the model chooses each command's timeout. Evidence: [core/harness/tools/shell.py:179](https://github.com/HKUDS/DeepCode/blob/84c37f79c726bd12e74e39541adceb8b5e47b0d5/core/harness/tools/shell.py#L179) (verified)
  - *To reach the next level:* Moderate default ceilings on steps, time and spend.
- **Cap:** G1: The iteration cap and Goal token budget exist but are off unless the operator sets them.

## Rule-of-Two check
[A] untrusted input: Public web pages via web_fetch (core/harness/tools/web.py:198-206), repository files and instruction files (core/harness/memory.py:463-476), MCP results · [B] sensitive data/systems: Any readable file outside the credential denylist (core/harness/tools/files.py:28-31) and provider keys in the shell environment (core/harness/env_sanitize.py:75-85) · [C] state change / egress: web_fetch auto-allowed as a read-only tool (core/harness/permissions.py:103-120); shell and writes behind approval · Same default session? Yes

## Highest-impact improvements
1. Require an explicit, confirmed operator action for any change that loosens approval or sandbox settings. (C2 D L1→L3, +0.100 before caps; Playbook 5)
2. Gate web_fetch behind approval (showing the full URL) once a session has read workspace or web content. (C5 B L1→L2, +0.050 before caps; Playbook 1)
3. Scrub credential-shaped variables from the shell, hook and code-mode environment by default. (C8 B L0→L1, +0.050 before caps; Playbook 4)
4. Ship default step and token ceilings for ordinary turns and cap model-chosen command timeouts. (C10 D L0→L2, +0.100 before caps; Playbook 3 step 3)
5. Show the full command and file diff in the TUI approval prompt. (C2 S L2→L3, +0.075 before caps; Playbook 5)

## Re-audit log
- C2 S: L3 → L2. The durable approval stores full arguments, but the TUI prompt (cli/tui/app.py:853-862) shows only tool name and reason and the tool card clips the command to the terminal width.
- C4 B: L1 → L0. bwrap binds / read-only (core/harness/sandbox.py:309-318), so the home directory and credential stores are readable, and the shell gets the full environment with provider keys (core/harness/tools/shell.py:203-211).
- C7 C: L1 → L0. The hash and signature hits in core/mcp and core/plugins are change detection, not integrity checks; no extension type is verified.
- C6 S: L3 → L2. Memory writes are gated but have no expiry, and security-relevant project configuration is not confirmed item by item.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the TUI with the default Ask preset on Linux or macOS with bubblewrap or Seatbelt present; the Windows Job Object backend (no write fence or network gate) and Linux hosts without bwrap run commands without a filesystem sandbox.
- The Desktop and Web approval cards were not examined in detail; they may show more of each call than the TUI.
- The legacy Paper2Code workflow runs its tools in an unattended legacy full_auto mode and was footnoted, not scored; external Codex and Claude Code sub-agent backends are separate products and were not scored.
- Full access, Read only, automations and Goals were reviewed only as far as they affect the default; the Read only preset trusts MCP servers' own read-only hints, as documented in docs/integrations/MCP.md.
