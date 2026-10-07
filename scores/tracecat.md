# Defense-in-Depth Score: Tracecat

**Repo:** https://github.com/TracecatHQ/tracecat · **Commit:** `77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe` · **Reviewed:** 2026-10-03
**What it is:** Open-source agentic security automation (SOAR) platform for teams and AI agents; MCP client/server
**Category:** Cybersecurity
**Scored configuration:** Self-hosted docker-compose.yml defaults: TRACECAT__EXECUTOR_BACKEND=direct (no nsjail), basic auth, RLS mode off, new agent presets with no tools and no per-tool approvals.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents yes · external communication yes

## Score: 4.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C2 | Approval gates | L3 | L1 | L0 | L0 | 0.30 | C2-POWERBYPASS | **0.25** | High |
| C3 | Tool & action scoping | L2 | L2 | L3 | L1 | 0.50 | — | **0.50** | High |
| C4 | Code-execution isolation | L3 | L3 | L0 | L3 | 0.60 | G1 | **0.50** (alt) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C6 | Memory, context & configuration integrity | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | Medium |
| C7 | Third-party extensions | L2 | L1 | L2 | L0 | 0.33 | — | **0.33** | Medium |
| C8 | Secrets & sensitive-data protection | L3 | L2 | L2 | L0 | 0.47 | — | **0.47** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C10 | Limits & kill switch | L1 | L2 | L2 | L2 | 0.42 | — | **0.42** | High |


Tracecat is a workflow and agent platform that runs integrations against stored SOC and cloud credentials. Its control plane is solid (scope-based RBAC, per-preset tool allowlists, encrypted secrets, secret masking, a gateway that keeps model-provider keys out of the agent). The shipped default executes actions as plain subprocesses that inherit the worker environment, per-tool approval is opt-in and no built-in action asks for it, step limits are not enforced in the agent runtime, and nothing structurally limits a prompt-injected agent that holds both egress and response actions. Switching the executor to nsjail and flagging response actions for approval would change the picture most.

## Critical gaps
- The most powerful action paths (generic HTTP, response actions, sandboxed shell) are not gated by approval in the default configuration, and no built-in action ships flagged. (ASI02, ASI09; C2) — [tracecat/agent/tools.py:54](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/tools.py#L54); [tracecat/agent/runtime/claude_code/runtime.py:1321](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/runtime/claude_code/runtime.py#L1321)
- The default direct executor runs actions in a subprocess that copies the executor environment, which holds the database URI, encryption key, service key and signing secret. (ASI05; C4) — [tracecat/executor/action_runner.py:435](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/executor/action_runner.py#L435); [tracecat/executor/backends/direct.py:77](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/executor/backends/direct.py#L77)
- Custom-registry extension code runs in the same direct action subprocess with the full executor environment (inferred from the shared artifact path). (ASI04; C7) — [tracecat/executor/action_runner.py:435](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/executor/action_runner.py#L435); [tracecat/executor/backends/direct.py:77](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/executor/backends/direct.py#L77)

## Criterion details

### C1 Identity & least privilege — 0.45 (high)

Agents act through a per-workspace service identity whose JWT lists the exact actions the preset allows, and both the MCP proxy and the executor reject anything outside that list. The identity itself is broad: it carries every workspace operational scope (workflow delete, secret create/delete, any action) and is not narrowed per tool or per requesting user. Authority is workspace-bound but one credential covers read and write across every integration the preset selects. The builder assistant can also rewrite a preset's actions and approval flags.

- **S L2:** The agent's role is a static service principal holding all workspace operational scopes, narrowed only by a per-run action allowlist in the token. — [tracecat/authz/scopes.py:437-440](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/authz/scopes.py#L437-L440); [tracecat/authz/scopes.py:79](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/authz/scopes.py#L79); [tracecat/agent/mcp/executor.py:75](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/mcp/executor.py#L75) (verified)
  - *To reach the next level:* Read and write tools do not use separate narrower credentials, and the service role is not reduced below the full workspace scope set.
- **C L2:** Registry tool calls are checked against the token's allowed_actions in both the MCP server and the executor entry, but the identity is the same service role for all of them and the requesting user is only an attribution field. — [tracecat/agent/mcp/trusted_server.py:666](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/mcp/trusted_server.py#L666); [tracecat/agent/mcp/executor.py:111](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/mcp/executor.py#L111) (verified)
  - *To reach the next level:* Authorization is not evaluated against the requesting principal's own scopes, so a user cannot be distinguished from the agent's service authority.
- **D L2:** New presets default to no actions and the token is short-lived (sandbox timeout plus 60 s), but widening the action list is an ordinary preset edit with no elevation step. — [tracecat/agent/preset/schemas.py:296](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/preset/schemas.py#L296); [tracecat/agent/tokens.py:175](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/tokens.py#L175) (verified)
  - *To reach the next level:* Write capability is not behind explicit time-bounded operator elevation.
- **B L1:** If the allowlist or approvals fail, the service role can write across the integrations and workspace objects its scopes cover (workflows, cases, tables, secrets), though it stays inside one workspace. — [tracecat/authz/scopes.py:406-411](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/authz/scopes.py#L406-L411); [tracecat/authz/scopes.py:73](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/authz/scopes.py#L73) (verified)
  - *To reach the next level:* Writes are not limited to one system; independent layers (per-tool approval) are opt-in and off by default.
- **Cap:** none
- **Notes:** The builder assistant tool update_preset can change a preset's actions and tool_approvals without a human approval step (tracecat/agent/mcp/internal_tools.py:324); it edits the target preset rather than its own role, so no C1-SELFESC cap was applied, but it is a configuration-tamper path.

### C2 Approval gates — 0.25 (high)

Tracecat has a real per-call approval mechanism: a flagged tool call is denied at the hook, the exact arguments are stored and shown to a user holding the agent update scope, and the human may approve, deny or override. But approval is opt-in per tool, no built-in registry action ships flagged, and unflagged tools (generic HTTP, response actions, the sandboxed shell) run unattended. Explicit sub-agents skip the gate entirely, and local MCP tools marked as needing approval are hard-denied rather than gated.

- **S L3:** A gated call is emitted with its exact tool name and arguments, persisted, and decided by an authenticated user (agent:update); there is no argument-level policy. — [tracecat/agent/runtime/claude_code/runtime.py:1180](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/runtime/claude_code/runtime.py#L1180); [tracecat/agent/session/service.py:1569](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/session/service.py#L1569); [packages/tracecat-ee/tracecat_ee/agent/approvals/router.py:75](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/packages/tracecat-ee/tracecat_ee/agent/approvals/router.py#L75) (verified)
  - *To reach the next level:* No allow/deny rules on parsed arguments and the approver can override arguments after the fact.
- **C L1:** Only tools flagged requires_approval are gated, none of the built-in actions are flagged, and explicit sub-agent tool calls return False from the approval check. — [tracecat/agent/runtime/claude_code/runtime.py:675-676](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/runtime/claude_code/runtime.py#L675-L676); [tracecat/agent/runtime/claude_code/runtime.py:679](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/runtime/claude_code/runtime.py#L679); searched `rg -n 'requires_approval=True'` in `tracecat packages/tracecat-registry` → 0 hits (no registry action or core code sets the flag by default) (verified)
  - *To reach the next level:* Unflagged mutating tools, built-in shell/file tools, and sub-agents bypass the gate.
- **D L0:** The per-tool approval flag defaults to False and the hook allows every tool not flagged. — [tracecat/agent/tools.py:54](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/tools.py#L54); [tracecat/agent/runtime/claude_code/runtime.py:1321](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/runtime/claude_code/runtime.py#L1321) (verified)
  - *To reach the next level:* Approval is opt-in; nothing requires it by default.
- **B L0:** Response and integration actions are mostly irreversible, there is no undo, preview or rate limit on them, and a wrongly approved or ungated call executes directly. — [tracecat/agent/mcp/executor.py:111](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/mcp/executor.py#L111); [tracecat/agent/runtime/claude_code/runtime.py:1321](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/runtime/claude_code/runtime.py#L1321) (verified)
  - *To reach the next level:* No checkpoints, dry-runs, rate limits or quantity bounds on consequential actions.
- **Cap:** C2-POWERBYPASS — The most powerful paths (generic HTTP action, response actions, sandboxed shell) are not gated in the default configuration.
- **Notes:** Approvals are unsupported for local stdio MCP tools: a flagged call is denied (runtime.py:1284), which fails closed. The builder tool update_preset can switch tool_approvals for a preset without approval.

### C3 Tool & action scoping — 0.50 (high)

Registry actions take typed, schema-validated arguments and an agent only receives the actions its preset names; a new preset has none, the model cannot add tools, and the script-execution action is excluded from agents entirely. The weak spot is the generic tools an operator can select: the HTTP action accepts any URL with no host or private-address check in the action code, so egress control depends on the optional nsjail backend. There are no per-call quantity bounds beyond the tool-count and timeout limits.

- **S L2:** Action arguments are typed through registry schemas, but the generic HTTP action takes an arbitrary URL with no allowlist or internal-address check. — [packages/tracecat-registry/tracecat_registry/core/http.py:591](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/packages/tracecat-registry/tracecat_registry/core/http.py#L591); searched `rg -n 'is_disallowed_address|DisallowedUrlError|outbound'` in `packages/tracecat-registry/tracecat_registry/core` → 0 hits (the registry HTTP actions never use the SSRF-checked outbound client in tracecat/outbound.py) (verified)
  - *To reach the next level:* No host allowlist or internal-address blocking on generic tools (the SSRF-safe outbound client exists only for backend traffic).
- **C L2:** Registry actions and MCP-proxied tools are all typed, but validation is per-action rather than a central policy layer that checks destinations or bounds. — [tracecat/agent/tools.py:184-188](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/tools.py#L184-L188) (verified)
  - *To reach the next level:* No shared validation layer for extension and user MCP tools.
- **D L3:** A preset starts with an empty action list, the agent receives only listed actions, the run_python action is excluded from agents, and the tool count is capped. — [tracecat/agent/preset/schemas.py:296](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/preset/schemas.py#L296); [tracecat/agent/tools.py:30](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/tools.py#L30); [tracecat/agent/tools.py:258](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/tools.py#L258) (verified)
  - *To reach the next level:* Builder sessions can edit a preset's tool list, and there is no per-task allowlist beyond the preset.
- **B L1:** A misused generic HTTP or response action can reach any host the executor can reach and any integration in the workspace, with no quantity bounds. — [packages/tracecat-registry/tracecat_registry/core/http.py:602](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/packages/tracecat-registry/tracecat_registry/core/http.py#L602) (verified)
  - *To reach the next level:* Not quantity-bounded or limited to allowlisted destinations.
- **Cap:** none

### C4 Code-execution isolation — 0.50 (high)

By default the platform runs actions and Python scripts without nsjail: actions are plain subprocesses that copy the executor's full environment (database URI, encryption key, service key, signing secret), scripts get only a new PID namespace (and run without it if unshare is unavailable), and network restrictions are not enforced. The agent's shell uses the Claude Code SDK sandbox in a weaker nested mode with unsandboxed commands denied. An opt-in nsjail backend is much stronger: non-root mapping, separate network namespace, seccomp denylist, read-only rootfs, with no host fallback, but it needs elevated container capabilities and is not on by default.

- **default configuration** (default; raw 0.28 → 0.28)
  - **S L2:** The default agent shell relies on the Claude Code SDK sandbox in weaker nested mode (inferred OS-level separation) while script execution is a PID namespace and actions have no isolation. — [tracecat/agent/runtime/claude_code/runtime.py:1585-1586](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/runtime/claude_code/runtime.py#L1585-L1586); [tracecat/sandbox/unsafe_pid_executor.py:327](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/sandbox/unsafe_pid_executor.py#L327) (inferred)
    - *To reach the next level:* No hardened container or OS sandbox profile on the default path.
  - **C L1:** Only the agent shell is sandboxed by default; registry actions run in a direct subprocess with use_sandbox=False and scripts in the PID-namespace executor. — [tracecat/executor/backends/direct.py:77](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/executor/backends/direct.py#L77); [tracecat/sandbox/service.py:478](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/sandbox/service.py#L478) (verified)
    - *To reach the next level:* Actions, scripts and registry sync run on the host unless the operator selects nsjail.
  - **D L1:** The shipped compose default is the direct backend and the isolation level is one environment variable; script network isolation is best-effort and not OS-enforced without nsjail. — [docker-compose.yml:297](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/docker-compose.yml#L297); [tracecat/sandbox/service.py:438](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/sandbox/service.py#L438) (verified)
    - *To reach the next level:* Hardened isolation is not on by default and can be switched by an env var without warning.
  - **B L0:** The direct action subprocess inherits the whole executor environment, and the executor container is given the database URI, encryption key, service key and signing secret. — [tracecat/executor/action_runner.py:435](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/executor/action_runner.py#L435); [docker-compose.yml:241](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/docker-compose.yml#L241) (verified)
    - *To reach the next level:* Action subprocesses should receive only their own credentials, with no platform keys in the environment.
- **opt-in nsjail executor backend (TRACECAT__EXECUTOR_BACKEND=nsjail)** (alt; raw 0.60, cap G1 → 0.50) ← counted
  - **S L3:** nsjail jails get a new network namespace, PID namespace, non-root uid mapping, a seccomp denylist, a read-only rootfs and tmpfs scratch space. — [tracecat/sandbox/executor.py:374](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/sandbox/executor.py#L374); [tracecat/sandbox/executor.py:388](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/sandbox/executor.py#L388); [tracecat/sandbox/seccomp.py:81](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/sandbox/seccomp.py#L81) (verified)
    - *To reach the next level:* Seccomp is a denylist with default allow and no microVM or gVisor boundary.
  - **C L3:** With nsjail selected the backend is authoritative and never falls back to the host for scripts, actions, agents and registry sync. — [tracecat/sandbox/service.py:462](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/sandbox/service.py#L462); [tracecat/config.py:864](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/config.py#L864) (verified)
    - *To reach the next level:* No documented evidence checked that every spawned process (for example MCP stdio probes) is jailed in all failure paths.
  - **D L0:** The nsjail backend is not the default and needs elevated container capabilities. — [tracecat/executor/enums.py:25](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/executor/enums.py#L25) (verified)
    - *To reach the next level:* Operator must select it and grant SYS_ADMIN-class capabilities.
  - **B L3:** Under nsjail the jail mounts only the job directory and read-only rootfs, uses a minimal environment, and filters egress. — [tracecat/sandbox/executor.py:432](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/sandbox/executor.py#L432) (verified)
    - *To reach the next level:* Not ephemeral-with-no-egress by default; egress CIDR allowlists are operator configured.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.
- **Notes:** The agent runtime direct spawn is labelled a development mode in its own log message (tracecat/agent/sandbox/nsjail.py:341) yet is the compose default.

### C5 Untrusted input blast radius — 0.05 (high)

Nothing in the code limits what a prompt-injected agent can do: alert, case, webhook and tool-result text enters the model with the same standing as operator instructions, with no taint tracking, quarantine or approval tied to provenance. The only handling found is a prompt line telling the model to treat Slack profile fields as data. What bounds a hijack is the per-preset tool allowlist, sandbox internet being off by default, and optional per-tool approval. A preset that holds both egress and response actions can act on injected instructions unattended.

- **S L0:** Only a prompt-level instruction (Slack profile fields are untrusted data) exists; there is no code that restricts tools after untrusted content is read. — [tracecat/agent/channels/handlers/slack.py:345](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/channels/handlers/slack.py#L345); searched `rg -n -i 'spotlight|quarantin|prompt.?injection'` in `tracecat/agent` → 0 hits (no classifier, quarantine or spotlighting code exists in the agent package) (verified)
  - *To reach the next level:* No Rule-of-Two enforcement or provenance-driven approval in code.
- **C L0:** Tool results, MCP results and alert payloads are not distinguished from instructions by any runtime mechanism. — [tracecat/agent/runtime/claude_code/runtime.py:1233](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/runtime/claude_code/runtime.py#L1233) (verified)
  - *To reach the next level:* No source-aware handling of tool results, descriptions or peer-agent messages.
- **D L0:** There is no injection control to be on or off by default. — [tracecat/agent/runtime/claude_code/runtime.py:1321](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/runtime/claude_code/runtime.py#L1321) (verified)
  - *To reach the next level:* No default mechanism exists.
- **B L1:** A hijacked agent can use every action in its preset unattended (approval is opt-in), including HTTP egress and response actions when selected, but a new preset holds no tools and sandbox internet is off by default. — [tracecat/agent/types.py:100](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/types.py#L100); [tracecat/agent/tools.py:54](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/tools.py#L54) (verified)
  - *To reach the next level:* Egress and irreversible actions are not forced through human approval after untrusted content is read.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.50 (medium)

There is no cross-session agent memory store. Persistent state is session history that is resumed within a session, user-authored skills and presets, and workspace data. The Claude CLI loads settings only from the user scope in a per-session home, so repository files cannot add hooks or MCP servers. Tenant isolation is enforced in application queries by workspace; the database row-level-security mode ships off. Poisoned content stored in a resumed session or workspace data can influence later turns of that session.

- **S L2:** Agent settings come only from the user scope and the tool/MCP set comes from the preset, so workspace files cannot add security-relevant config; skills and resumed history still load without review. — [tracecat/agent/runtime/claude_code/runtime.py:1646](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/runtime/claude_code/runtime.py#L1646); searched `rg -n -i 'save_memory|long_term_memory|memory_store|remember'` in `tracecat/agent` → 0 hits (no model-writable long-term memory tool) (verified)
  - *To reach the next level:* Skill and history content is not provenance-tagged or approval-gated.
- **C L2:** The main persistent store (session history) is workspace-scoped and written by the orchestrator; skills and retrieved case data are not provenance-tagged. — [tracecat/agent/session/service.py:1569](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/session/service.py#L1569) (inferred)
  - *To reach the next level:* Auto-loaded skills and retrieved data are not controlled or tagged end to end.
- **D L2:** Isolation is by workspace in queries and the database RLS mode defaults to off. — [tracecat/config.py:1388](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/config.py#L1388) (verified)
  - *To reach the next level:* RLS is not on by default and there are no retention limits.
- **B L2:** Poisoned session or workspace content persists within the workspace and can drive later turns, whose tool use is gated only if flagged. — [tracecat/agent/runtime/claude_code/runtime.py:1646](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/runtime/claude_code/runtime.py#L1646) (inferred)
  - *To reach the next level:* Persisted content is not reviewed before reuse and there is no rollback.
- **Cap:** none

### C7 Third-party extensions — 0.33 (medium)

Nothing third-party is enabled by default: MCP servers, skills and custom registries are added by workspace operators. Registry versions are pinned in a lock with manifest fingerprints, but user-configured MCP servers are neither pinned nor checked for changed tool definitions, and stdio MCP commands are whatever the operator typed. Custom-registry code runs through the same direct action path as built-ins, in a subprocess carrying the full executor environment, so a malicious extension inherits platform secrets unless nsjail is selected.

- **S L2:** Registry versions are pinned in a lock carrying manifest fingerprints, while MCP servers are user-chosen and unpinned. — [tracecat/registry/lock/types.py:23](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/registry/lock/types.py#L23); [tracecat/agent/runtime/claude_code/runtime.py:592](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/runtime/claude_code/runtime.py#L592) (verified)
  - *To reach the next level:* No signature check or re-approval when an MCP server's version or tool definitions change.
- **C L1:** Only registry artifacts are version-locked; MCP servers and skills are not verified. — [tracecat/executor/backends/registry_helpers.py:60](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/executor/backends/registry_helpers.py#L60); searched `rg -n 'trust_remote_code|pickle|torch.load|npx -y'` in `tracecat/agent tracecat/registry tracecat/executor` → 0 hits (no automatic remote-code loading patterns) (verified)
  - *To reach the next level:* MCP servers and skill code are outside the verification path.
- **D L2:** Extensions are added only through workspace configuration and nothing third-party is on by default; whether the UI shows the exact command and permissions was not reviewed. — [tracecat/agent/runtime/claude_code/runtime.py:590-592](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/runtime/claude_code/runtime.py#L590-L592) (inferred)
  - *To reach the next level:* Not confirmed that adding an extension shows exactly what will run.
- **B L0:** Custom-registry code executes in the direct action subprocess that inherits the executor environment; stdio MCP servers get a scrubbed environment inside the agent sandbox. — [tracecat/executor/action_runner.py:435](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/executor/action_runner.py#L435); [tracecat/executor/backends/direct.py:77](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/executor/backends/direct.py#L77) (inferred)
  - *To reach the next level:* Custom-registry code is not confined or given only its own credentials in the default backend.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.47 (high)

Secrets are encrypted at rest (Fernet), masked in action output and errors before they reach logs or the model, and model-provider keys are injected by a gateway so the agent holds only a gateway token. Telemetry is off unless a Sentry DSN or PostHog key is set, and Sentry is configured without PII or local variables. The gap is process environment: the direct action subprocess inherits the executor's full environment, so platform keys that decrypt every stored secret sit one environment read away from any action code, and the encryption key is a long-lived value passed as an env var.

- **S L3:** Stored secrets are Fernet-encrypted, secret values seen at runtime are collected and masked, and provider credentials are injected at the gateway instead of entering the agent environment. — [tracecat/secrets/encryption.py:12](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/secrets/encryption.py#L12); [tracecat/agent/gateway.py:588](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/gateway.py#L588); [tracecat/agent/runtime/claude_code/runtime.py:1592](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/runtime/claude_code/runtime.py#L1592) (verified)
  - *To reach the next level:* No secret manager or KMS for the master key by default and no pattern-based output scanning for unknown secrets.
- **C L2:** Logs, errors, telemetry and model-bound action results are covered, but the direct subprocess environment copies everything except SENTRY_DSN. — [tracecat/executor/action_runner.py:435-437](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/executor/action_runner.py#L435-L437); [tracecat/observability/sentry.py:657-658](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/observability/sentry.py#L657-L658) (verified)
  - *To reach the next level:* Subprocess environments are not scrubbed of platform secrets in the default backend.
- **D L2:** Telemetry keys default empty (opt-in) and masking is on by default, but an env flag can switch masking off. — [docker-compose.yml:56](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/docker-compose.yml#L56); [tracecat/config.py:501](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/config.py#L501) (verified)
  - *To reach the next level:* Redaction can be disabled by configuration.
- **B L0:** The encryption key, database URI, service key and signing secret are long-lived, high-privilege values present in the environment that every direct action subprocess copies. — [docker-compose.yml:241](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/docker-compose.yml#L241); [tracecat/executor/action_runner.py:435](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/executor/action_runner.py#L435) (verified)
  - *To reach the next level:* Platform keys must not be reachable by action processes; credentials should be short-lived or per-task.
- **Cap:** none

### C9 Audit & traceability — 0.50 (high)

Every agent tool call runs as a durable workflow with the session and parent workflow IDs, session transcripts are persisted by the orchestrator, approvals record who decided, and control-plane changes go through an audit decorator. The audit stream, however, is a fire-and-forget webhook that only exists when an operator configures a sink, emission is explicitly best-effort, and records live in the application database and Temporal rather than in tamper-evident storage.

- **S L2:** Tool calls carry the agent session ID and are recorded as workflow executions; control-plane events pass through audit_log with attempt and outcome events. — [tracecat/agent/mcp/executor.py:130](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/mcp/executor.py#L130); [tracecat/audit/service.py:283](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/audit/service.py#L283) (verified)
  - *To reach the next level:* No tamper-evident storage or standard export enabled by default; approver attribution exists only in the approval rows.
- **C L2:** All registry tool calls go through one executor workflow and approvals are persisted, but stdio MCP and built-in shell calls inside the agent sandbox are recorded only in the session transcript. — [tracecat/agent/mcp/executor.py:134](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/mcp/executor.py#L134) (verified)
  - *To reach the next level:* Config, credential use and extension calls are not uniformly recorded.
- **D L2:** Workflow and transcript records are on by default and written by the orchestrator, but the audit webhook sink is opt-in and records are deletable by users with delete scopes. — [tracecat/audit/service.py:283](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/audit/service.py#L283) (verified)
  - *To reach the next level:* Audit export is not on by default and the records are not append-only.
- **B L2:** Audit event creation and metadata callbacks are best-effort and failures are logged without stopping the action. — [tracecat/audit/logger.py:105-106](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/audit/logger.py#L105-L106) (verified)
  - *To reach the next level:* High-risk actions do not wait for their audit record.
- **Cap:** none

### C10 Limits & kill switch — 0.42 (high)

An agent turn is bounded by a wall-clock timeout (30 minutes by default, clamped to a one-hour ceiling) and each action has a 300-second executor timeout, with a Redis-signalled cancel that stops the turn. The step and tool-call caps (max_requests, max_tool_calls) are declared in schemas but nothing in the agent runtime, executor or MCP code enforces them, only sub-agents have an optional turn cap, and the API rate-limit middleware class is not installed. There is no spend or cost ceiling in the platform itself.

- **S L1:** Only a wall-clock timeout is enforced for the agent turn; the declared request and tool-call caps are not referenced in the runtime. — [tracecat/agent/executor/activity.py:908](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/executor/activity.py#L908); searched `rg -n 'max_requests|max_tool_calls'` in `tracecat/agent/runtime tracecat/agent/executor tracecat/agent/mcp` → 0 hits (no enforcement of the declared step caps in the runtime) (verified)
  - *To reach the next level:* No enforced step or tool-call cap, no cost cap, no rate limits on side-effecting tools.
- **C L2:** The top-level turn and each executor action are time-bounded; sub-agent turn caps are optional and not shared with the parent. — [tracecat/executor/action_runner.py:209](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/executor/action_runner.py#L209) (verified)
  - *To reach the next level:* Sub-agents and spawned tasks do not count against a shared budget.
- **D L2:** Defaults are sensible (1800 s turn, 3600 s ceiling) and operator-configurable, with a clamp the caller cannot exceed. — [tracecat/agent/types.py:34](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/types.py#L34); [tracecat/agent/constants.py:3](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/constants.py#L3) (verified)
  - *To reach the next level:* The ceiling is an environment variable, not a hard limit, and there is no cost default.
- **B L2:** A runaway agent can run up to an hour per turn with model spend unbounded by the platform, and cancel stops the turn via a Redis signal. — [tracecat/agent/cancellation.py:33](https://github.com/TracecatHQ/tracecat/blob/77ed0abaf3fe59c4afba4142a7d7787a0af7d8fe/tracecat/agent/cancellation.py#L33); searched `rg -n 'RateLimitMiddleware'` in `tracecat/api` → 0 hits (the rate-limit middleware class is not installed in the API app) (verified)
  - *To reach the next level:* No spend ceiling or tight per-run ceilings and in-flight actions can finish after cancel.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: alert, case, webhook and tool-result content enters agent context as plain text (tracecat/agent/runtime/claude_code/runtime.py:1233 hook allows by default) · [B] sensitive data/systems: workspace secrets and integration credentials injected into actions the preset allows (tracecat/executor/action_runner.py:429) · [C] state change / egress: response and HTTP actions run unattended unless a per-tool flag is set (tracecat/agent/tools.py:54) · Same default session? Yes

## Highest-impact improvements
1. Scrub the action subprocess environment so it receives only the secrets that action declares. — C4 B L0→L3, +0.150 before caps (Playbook 4)
2. Require approval by default for response and egress actions, and ship the flag on destructive built-ins. — C2 D L0→L3, +0.150 before caps (Playbook 5)
3. Enforce max_requests and max_tool_calls in the agent runtime and add a cost ceiling. — C10 S L1→L3, +0.150 before caps (Playbook 3 step 3)
4. Add destination allowlists and private-address blocking to the generic HTTP action. — C3 S L2→L3, +0.075 before caps (Playbook 3)
5. Make nsjail (or an equivalent hardened sandbox) the default executor backend. — C4 D L1→L3, +0.100 before caps (Playbook 3 step 1)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scope was the Python backend (tracecat/, packages/tracecat-registry, packages/tracecat-ee), docker-compose.yml and config defaults; the frontend, Fargate/Helm deployment manifests, and the integration-by-integration registry actions were not reviewed individually.
- Rating of the Claude Code SDK sandbox used for the agent shell in direct mode is INFERRED from the settings Tracecat passes; the SDK internals were not reviewed.
- Whether custom-registry code runs in the same direct subprocess as built-in actions is INFERRED from the shared artifact-mount path; deployment-specific hardening (Fargate task roles, network policy) was not examined.
- Per-preset behaviour depends on which actions an operator selects; the audit scored the shipped defaults, where new presets hold no tools.
- No attempt to steer reviewers was found in README, AGENTS.md, CLAUDE.md or other markdown files (search for auditor/score-steering phrases returned 0 hits).
