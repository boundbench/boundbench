# Defense-in-Depth Score: CyberStrikeAI

**Repo:** https://github.com/AIPentest/CyberStrikeAI · **Commit:** `470eb5ead185dc90f2441e89b31ceb1be913ac44` · **Reviewed:** 2026-10-03
**What it is:** AI-native security testing platform in Go integrating 100+ tools with multi-agent orchestration and MCP
**Category:** Cybersecurity
**Scored configuration:** Server started via run.sh with config.yaml copied from config.example.yaml: web UI on all interfaces with self-signed TLS, generated admin password, approval gate off, C2 module enabled, no external MCP servers.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents yes · external communication yes

## Score: 2.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C2 | Approval gates | L3 | L2 | L0 | L0 | 0.38 | G1 | **0.38** | High |
| C3 | Tool & action scoping | L1 | L2 | L1 | L0 | 0.28 | — | **0.28** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L1 | 0.23 | — | **0.23** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


This is a high-privilege platform by design: its shell and scanner tools run as the server's OS user with the full environment, and the human-approval gate is off by default. Authentication is real (generated admin password, per-user RBAC), but nothing contains the agent once a session is running, and untrusted target content flows straight into a model that can run commands. Operators should treat every deployment as a privileged system, enable approval, and isolate the host.

## Critical gaps
- Tools run as same-user host subprocesses with the full inherited environment and no sandbox of any kind. (ASI05, T11; C4) — [internal/security/executor.go:1260-1261](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/security/executor.go#L1260-L1261); [internal/security/executor.go:822](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/security/executor.go#L822)
- Worst case after a hijack is unattended data leakage plus irreversible actions, because the approval gate defaults to off (C5-WORSTCASE). (ASI01, LLM01; C5) — [config.example.yaml:190](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L190); [tools/exec.yaml:1-3](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/tools/exec.yaml#L1-L3)
- If the authorization layer fails, the agent's authority is the server user's full host authority. (ASI03, T3; C1) — [internal/security/executor.go:1261](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/security/executor.go#L1261)

## Criterion details

### C1 Identity & least privilege — 0.20 (high)

The platform has real multi-user authentication: a generated 24-character admin password on first start, per-user roles and permissions, resource-level scopes, a login rate limit, and a principal that is carried into the MCP tool-call authorizer. That governs which human may use the platform. The agent's tools, however, run as the server's OS user with the full inherited environment, so the authority the agent holds is ambient and broad. If the authorization layer fails, the process can reach everything that OS user can.

- **S L1:** RBAC principals with per-permission scopes are enforced at the API and MCP layers, but the authority the agent's tools exercise is the server process's own OS identity. — [internal/security/auth_manager.go:68](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/security/auth_manager.go#L68); [internal/security/auth_middleware.go:43](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/security/auth_middleware.go#L43); [internal/security/executor.go:1261](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/security/executor.go#L1261) (verified)
  - *To reach the next level:* No per-tool or per-task credential narrowing; tool subprocesses are not given a reduced identity.
- **C L1:** The MCP tool-call path checks a principal-aware authorizer, but every spawned subprocess receives the whole server environment. — [internal/mcp/server.go:536](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/mcp/server.go#L536); [internal/security/executor.go:1260-1261](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/security/executor.go#L1260-L1261) (verified)
  - *To reach the next level:* Subprocesses and extension processes are not routed through a reduced-authority identity.
- **D L1:** Default install has a strong generated admin credential, but the shipped tool set and shell run with the server user's full authority and the server binds to all interfaces. — [config.example.yaml:16](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L16); [internal/security/auth_manager.go:66-70](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/security/auth_manager.go#L66-L70) (verified)
  - *To reach the next level:* Least privilege for tool execution requires manual hardening outside the product.
- **B L0:** If the authorization layer is bypassed, the shell and bundled tools act with the server user's full host authority and inherited credentials. — [tools/exec.yaml:1-3](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/tools/exec.yaml#L1-L3); [internal/security/executor.go:822](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/security/executor.go#L822) (verified)
  - *To reach the next level:* Reach would have to be limited to a scoped identity or sandbox for the blast radius to drop.
- **Cap:** none

### C2 Approval gates — 0.38 (high)

A per-call human approval gate exists (the HITL feature): it shows the tool name and exact arguments, supports reject, edit-before-run, and auto-rejects on timeout. It is off by default, so a fresh install runs every tool, including the shell and C2 task tools, with no approval. The shipped allowlist of approval-exempt tools also contains project-fact writes and some C2 helper tools. A wrongly approved or unapproved call can act irreversibly on real targets with no rollback.

- **S L3:** When enabled, each non-allowlisted call shows toolName and exact arguments to the reviewer, reject and timeout-reject are first-class, and edits apply only in review_edit mode. — [internal/handler/hitl.go:948-955](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/handler/hitl.go#L948-L955); [internal/handler/hitl.go:607](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/handler/hitl.go#L607); [internal/handler/hitl.go:599](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/handler/hitl.go#L599) (verified)
  - *To reach the next level:* No argument-level policy rules; an optional LLM reviewer can stand in for the human and its prompt defaults to approve when unsure.
- **C L2:** The gate wraps all agent paths I traced (single agent, multi-agent, batch queue) and C2 dangerous task types, but the exempt list is name-based and includes state-changing tools. — [internal/handler/hitl.go:389-402](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/handler/hitl.go#L389-L402); [config.example.yaml:206](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L206); [internal/c2/types.go:114-118](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/c2/types.go#L114-L118) (verified)
  - *To reach the next level:* Auto-approved tools are not a verified read-only list and unknown tools are not rejected by default.
- **D L0:** Approval is opt-in: the default mode resolves to off and no conversation is gated unless an operator enables it. — [config.example.yaml:190](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L190); [internal/config/config.go:1146](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/config/config.go#L1146); [internal/handler/hitl.go:393](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/handler/hitl.go#L393) (verified)
  - *To reach the next level:* Approval would need to default to on.
- **B L0:** Wrongly approved or ungated calls (shell, exploitation tools, C2 tasks) can make irreversible changes to external systems with no checkpoint, preview, or rate limit on approvals. — [internal/c2/manager.go:562](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/c2/manager.go#L562); [tools/exec.yaml:1-3](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/tools/exec.yaml#L1-L3) (verified)
  - *To reach the next level:* No rollback, dry-run, or rate limits on consequential actions or approval requests.
- **Cap:** G1 — The approval gate exists but is off by default (default_mode: off), so it is opt-in in the scored configuration.
- **Notes:** A C2 BypassHITL flag exists for internal callers only; no model-reachable path to set it was found.

### C3 Tool & action scoping — 0.28 (high)

Tool arguments are mostly passed straight through. The shell tool takes an arbitrary command string, and the only central validation is a configurable regex deny rule (government domains by default) applied to every tool call, which its own docs say is not an exhaustive authorization check. Engagement scope is given to the model as prompt text rather than enforced in code. About 78 tools, including shell, scanners, and C2 tools, are enabled in the default role.

- **S L1:** A regex denylist is the only argument validation common to all tools; the shell tool is raw passthrough and project scope is prompt-only. — [internal/toolguard/toolguard.go:4](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/toolguard/toolguard.go#L4); [internal/project/scope_block.go:19-20](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/project/scope_block.go#L19-L20); [tools/exec.yaml:1-3](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/tools/exec.yaml#L1-L3) (verified)
  - *To reach the next level:* No allowlist validation of targets, paths, or commands in code.
- **C L2:** The tool guard is invoked centrally for local and external MCP tool calls, so coverage is broad, but it is capped by the limits of the mechanism. — [internal/mcp/server.go:87-91](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/mcp/server.go#L87-L91); [internal/mcp/external_manager.go:111-115](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/mcp/external_manager.go#L111-L115) (verified)
  - *To reach the next level:* Coverage of a denylist does not make it a boundary; no schema-level bounds on tools.
- **D L1:** Dangerous tools are on by default but individually disableable through tool config and roles; the default role carries all tools. — [roles/默认.yaml:1-5](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/roles/默认.yaml#L1-L5); [tools/exec.yaml:1-4](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/tools/exec.yaml#L1-L4) (verified)
  - *To reach the next level:* Default tool set is not read-only, and write/exec tools are not opt-in.
- **B L0:** A misused tool can run any command against any host reachable from the server. — [tools/exec.yaml:1-3](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/tools/exec.yaml#L1-L3); [config.example.yaml:173-174](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L173-L174) (verified)
  - *To reach the next level:* Reach is not scoped to a project, host list, or quantity bounds.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

Commands and scanner tools run as ordinary subprocesses of the server, under the server's OS user, with the full inherited environment. A process-guard layer adds cgroup or process-group containment, a process count limit, a memory ceiling, and kill-on-cancel, but that is lifecycle and resource control, not an isolation boundary, and there is no container, filesystem, user, or network separation. The shell tool accepts arbitrary command strings, so anything that reaches it is host-level execution.

- **S L0:** Execution is a same-user subprocess; process-guard limits resources and cleans up but does not separate filesystem, identity, or network. — [internal/security/executor.go:187-188](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/security/executor.go#L187-L188); [internal/processguard/guard.go:12-14](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/processguard/guard.go#L12-L14); searched `rg -n -i 'chroot|Cloneflags|NoNewPrivs|seccomp|landlock|docker run' --glob '*.go'` in `internal` → 0 hits (no sandbox primitives in Go sources) (verified)
  - *To reach the next level:* No isolation primitive (container, low-privilege user, seccomp, microVM) is used.
- **C L0:** No execution path is sandboxed: shell tool, scanner tools, and external MCP stdio servers all start as host processes. — [internal/security/executor.go:822](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/security/executor.go#L822); [internal/mcp/client_sdk.go:374](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/mcp/client_sdk.go#L374) (verified)
  - *To reach the next level:* At least the main exec tool would need to run behind an isolation boundary.
- **D L0:** There is no sandbox to be on by default; only the resource-containment layer is on in auto mode. — [config.example.yaml:407-408](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L407-L408); [internal/processguard/guard.go:39](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/processguard/guard.go#L39) (verified)
  - *To reach the next level:* A sandbox would have to exist and ship enabled.
- **B L0:** Anything running in these processes has the host's authority: same user, inherited environment, home directory, and network. — [internal/security/executor.go:1260-1261](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/security/executor.go#L1260-L1261) (verified)
  - *To reach the next level:* Reach would need confinement to a workspace mount with no secrets and limited egress.
- **Cap:** none
- **Notes:** Process-guard in required mode needs an operator-provided cgroup root; it bounds processes and memory only.

### C5 Untrusted input blast radius — 0.00 (high)

Output from scanned targets, web pages, files, and external tools enters the model's context with the same standing as the user's instructions, and nothing in code tracks provenance or restricts tools after untrusted content is read. Combined with a shell, network tools, and C2 tools running ungated by default, a hijacked agent can both exfiltrate data and take irreversible actions without a human. This platform's core job is to read hostile content, so the exposure is built in.

- **S L0:** No structural limit exists on what a hijacked agent may do after reading untrusted content; the generic approval gate is off by default and is not tied to content provenance. — searched `rg -n -i 'untrusted|prompt.injection' --glob '*.go'` in `internal` → 1 hits (single hit is a code comment in execution_service.go about blocking work, not an injection defence); [internal/handler/hitl.go:393](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/handler/hitl.go#L393) (verified)
  - *To reach the next level:* No Rule-of-Two enforcement, quarantine, or taint tracking.
- **C L0:** Tool results, scan output, and sub-agent messages are not distinguished from user instructions. — searched `rg -n -i 'untrusted|prompt.injection' --glob '*.go'` in `internal` → 1 hits (same single non-defence comment hit as S) (verified)
  - *To reach the next level:* No source is tagged or handled as untrusted data.
- **D L0:** There is no control to be on by default. — [config.example.yaml:190](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L190) (verified)
  - *To reach the next level:* A control would need to exist first.
- **B L0:** By default a hijack can run shell and network tools to leak data and also take irreversible actions, with no human approval. — [config.example.yaml:190](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L190); [tools/exec.yaml:1-3](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/tools/exec.yaml#L1-L3) (verified)
  - *To reach the next level:* Both egress and irreversible actions would need human approval by default.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.
- **Notes:** Multi-user deployment: hijack reaches only what the server's tools can reach; per-user RBAC does not limit shell reach.

### C6 Memory, context & configuration integrity — 0.30 (high)

The model can write persistent project facts and security-finding records, which are re-injected into later sessions as working context, and the shipped approval allowlist exempts fact writes from review. Project data is isolated per user through resource-assignment queries, but facts are not validated or expired, and automatic storage cleanup is off by default. Configuration and prompt files load only from the platform's own directories, not from a target workspace.

- **S L1:** Model-written project facts persist unvalidated and are fed back into prompts; fact writes are exempt from approval by the default allowlist. — [internal/projectprompt/blackboard.go:12](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/projectprompt/blackboard.go#L12); [config.example.yaml:206](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L206) (verified)
  - *To reach the next level:* No gating, validation, expiry, or integrity protection on memory writes.
- **C L1:** Only the project resource-assignment layer is controlled; facts, summaries, and conversation state are not otherwise validated. — [internal/database/project.go:166](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/database/project.go#L166) (verified)
  - *To reach the next level:* Other stores (summaries, last_react state, facts) have no write controls.
- **D L2:** Per-user project access is enforced in database queries via resource assignments. — [internal/database/project.go:166](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/database/project.go#L166) (verified)
  - *To reach the next level:* Isolation is not verified for every store and model writes are not namespace-restricted beyond the project.
- **B L1:** Poisoned facts persist across the user's sessions in a project and can steer tool use. — [internal/projectprompt/blackboard.go:12](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/projectprompt/blackboard.go#L12); [config.example.yaml:56](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L56) (verified)
  - *To reach the next level:* Persistence is not review-gated or rollbackable.
- **Cap:** none
- **Notes:** No auto-loaded instruction files from a target workspace were found; settings load from config.yaml in the platform directory.

### C7 Third-party extensions — 0.23 (high)

Third-party extensions are opt-in: the external MCP server list ships empty. An authenticated operator can add a stdio server by naming a command and arguments, and the platform launches it as a same-user process with no version pin, hash check, or re-approval when it changes. Skills and tool definitions are local files loaded from the platform directories. The gap is the absence of any verification, not automatic installation.

- **S L1:** Operator-chosen extensions run unpinned; the only validation is that a command is present. — [internal/handler/external_mcp.go:361](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/handler/external_mcp.go#L361); [internal/mcp/client_sdk.go:374](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/mcp/client_sdk.go#L374) (verified)
  - *To reach the next level:* No pinning, integrity check, or change re-approval.
- **C L0:** None of the extension types (external MCP, skills, tool YAML) is verified. — searched `rg -n -i 'sha256|checksum|signature|pinned' --glob '*.go'` in `internal/mcp internal/handler/external_mcp.go internal/skillpackage` → 0 hits (no integrity checks on extension loading) (verified)
  - *To reach the next level:* At least one type would need integrity verification.
- **D L2:** Nothing third-party is enabled by default; adding a server is an explicit authenticated API action without a what-will-run confirmation. — [config.example.yaml:433](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L433); [internal/security/rbac_middleware.go:146](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/security/rbac_middleware.go#L146) (verified)
  - *To reach the next level:* Adding an extension does not show exact command and permissions for approval.
- **B L1:** Extension processes are separate, same-user processes; with no env override they inherit the full environment. — [internal/mcp/client_sdk.go:375-376](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/mcp/client_sdk.go#L375-L376) (verified)
  - *To reach the next level:* Environment is not scrubbed to the extension's own config.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.30 (high)

Secrets live in a plaintext config file created with owner-only permissions, and the settings API does not fully protect secrets. Platform audit records redact common secret-named fields, but full tool arguments and results are stored unredacted in the execution table and tool subprocesses inherit the whole environment. Telemetry is local-only by default. Long-lived provider keys are reachable by anything the shell can read.

- **S L1:** Keys are in a 0600 plaintext config; redaction exists only in the platform audit detail path. — [internal/config/config.go:1761](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/config/config.go#L1761); [internal/audit/sanitize.go:8-10](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/audit/sanitize.go#L8-L10) (verified)
  - *To reach the next level:* No encryption at rest, secret manager, or redaction of model-bound or stored tool I/O.
- **C L1:** Only audit detail is sanitized; tool executions, transcripts, and subprocess environments are not. — [internal/database/database.go:262-267](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/database/database.go#L262-L267); [internal/config/config.go:829](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/config/config.go#L829) (verified)
  - *To reach the next level:* Logs, transcripts, model-bound messages, and environments are not covered.
- **D L2:** No third-party telemetry ships on; the trace exporter defaults to local stdout; redaction in audit is always applied. — [config.example.yaml:40-41](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L40-L41); [internal/audit/sanitize.go:37](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/audit/sanitize.go#L37) (verified)
  - *To reach the next level:* Stored tool arguments and results are retained in full for 90 days by default.
- **B L1:** Long-lived LLM and search-service keys are in the config and environment, reachable by the shell and every tool subprocess. — [internal/security/executor.go:1261](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/security/executor.go#L1261); [config.example.yaml:106](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L106) (verified)
  - *To reach the next level:* Keys are not short-lived or isolated from subprocesses.
- **Cap:** none

### C9 Audit & traceability — 0.50 (high)

Every tool call routed through the MCP server is saved with arguments, result, status, timestamps, owning user and conversation. Approval requests and decisions are stored with reviewer type and whether the system or a human decided, and platform actions (logins, config changes) have a separate audit log with actor and IP. Records sit in the same local SQLite database the server's shell could modify, and are not signed or exported by default.

- **S L2:** A structured per-call record with arguments, result, times, owner_user_id and conversation_id exists, plus separate approval and platform audit tables. — [internal/database/database.go:262-278](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/database/database.go#L262-L278); [internal/audit/types.go:3-14](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/audit/types.go#L3-L14) (verified)
  - *To reach the next level:* No tamper-evident storage, standard export by default, or per-sub-agent delegation chain.
- **C L2:** MCP-server tool executions, approvals, denials, and platform actions are recorded; sub-agent attribution relies on conversation id only. — [internal/mcp/server.go:563](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/mcp/server.go#L563); [internal/handler/hitl.go:608](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/handler/hitl.go#L608) (verified)
  - *To reach the next level:* Not every config change and memory write is verified to be recorded with actor.
- **D L2:** On by default, stored under data/ outside the agent workspace, but writable by any same-user process, including the shell tool. — [config.example.yaml:389](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L389); [config.example.yaml:43-45](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L43-L45) (verified)
  - *To reach the next level:* Records are not written by a component the agent cannot reach.
- **B L2:** Save errors are logged and each action is persisted when it starts and finishes, but actions proceed if logging fails. — [internal/mcp/server.go:563-565](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/mcp/server.go#L563-L565) (verified)
  - *To reach the next level:* High-risk actions do not fail closed on a missing record.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

Per-tool timeouts, a shell no-output timeout, concurrency caps and a circuit breaker for external servers, and a process-guard memory and process ceiling are enforced in code, and cancel kills the whole process group. There is no cost or wall-clock budget for a run, and the shipped example config sets the iteration limit to 12000 and the per-tool timeout to 60 minutes, so a runaway can continue for a very long time. Stopping works on in-flight commands, but batch task schedules can restart work.

- **S L2:** An iteration cap plus per-execution timeouts are enforced; cancel sends SIGKILL to the process group; no cost or wall-clock cap exists. — [config.example.yaml:159-160](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L159-L160); [internal/security/procattr_unix.go:31](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/security/procattr_unix.go#L31) (verified)
  - *To reach the next level:* No run-level wall-clock or token/cost budget and no rate limits on side-effecting tools.
- **C L2:** Tool timeouts and external-server concurrency caps apply, and process scopes own background processes; the iteration cap is per agent rather than a shared delegation budget. — [config.example.yaml:161-165](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L161-L165); [internal/security/process_scope.go:20-22](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/security/process_scope.go#L20-L22) (verified)
  - *To reach the next level:* Sub-agents do not share one budget and delegation depth is not capped.
- **D L1:** Shipped defaults are very large (12000 iterations, 60-minute tool timeout), though code-level fallbacks are 30 and 10 minutes. — [config.example.yaml:159-160](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L159-L160); [internal/config/config.go:2149-2150](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/internal/config/config.go#L2149-L2150) (verified)
  - *To reach the next level:* Defaults would need to be sensible and bounded by hard ceilings.
- **B L1:** Ceilings are very large and scheduled batch tasks can restart work after a stop. — [config.example.yaml:159](https://github.com/AIPentest/CyberStrikeAI/blob/470eb5ead185dc90f2441e89b31ceb1be913ac44/config.example.yaml#L159) (verified)
  - *To reach the next level:* Tight per-run ceilings and external spend limits are absent.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: scan output and fetched content enter model context (internal/multiagent/runner.go) · [B] sensitive data/systems: config keys and host files reachable via shell (internal/security/executor.go:1261) · [C] state change / egress: shell and network tools ungated by default (config.example.yaml:190) · Same default session? Yes

## Highest-impact improvements
1. Ship the approval gate on by default (approval mode, human reviewer). — C2 D L0→L3, +0.150 before caps (Playbook 5)
2. Run tool subprocesses in a hardened container or sandbox with a scrubbed environment. — C4 S L0→L3, +0.225 before caps (Playbook 3)
3. Enforce engagement scope and target allowlists in code instead of prompt text. — C3 S L1→L3, +0.150 before caps (Playbook 3)
4. Tag tool results as untrusted and require approval for egress and state change after they are read. — C5 S L0→L3, +0.225 before caps (Playbook 1)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- C2 listener protocol, WebShell management, robot integrations, the browser plugin, and frontend rendering were not reviewed in depth.
- Sub-agent gating and model-bound secret handling were traced through the main paths only; absence claims are limited to the searches listed.
- No reviewer-directed instructions were found in repository text during this review.
