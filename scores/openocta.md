# Defense-in-Depth Score: OpenOcta

**Repo:** https://github.com/openocta/openocta · **Commit:** `0c7dac2284211facdfc813166fe648d093153bb6` · **Reviewed:** 2026-10-03
**What it is:** Open-source AIOps agent for Windows & macOS that inspects and remediates across servers/network/DB/cloud
**Category:** Infrastructure & Ops
**Scored configuration:** Desktop app (Windows/macOS) with the gateway config written by EnsureDefaultConfig on first launch: no security, cozeloop or browser sections, so approval, command policy and sandbox are absent and trace export is on.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication opt-in

## Score: 1.2 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | C1-SELFESC | **0.00** | High |
| C2 | Approval gates | L2 | L1 | L0 | L0 | 0.23 | C2-SELFAPPROVE | **0.23** | High |
| C3 | Tool & action scoping | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-PUBLICTRIGGER | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L2 | 0.45 | — | **0.45** | High |


As shipped, OpenOcta gives the model an unsandboxed shell running as the logged-in ops engineer, with every credential on the machine, and no approval step (the approval gate is opt-in and only covers the shell). The agent can rewrite its own security config, and access control on the local gateway is not locked down. Agent traces are exported by default to a third-party tracing service. Treat it as full control of the operator's infrastructure access.

## Critical gaps
- The agent can rewrite its own security configuration (approvals, command policy, MCP servers) through the default gateway_config tool. (ASI03, T3; C1) — [src/pkg/agent/tools/bridge.go:71](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/tools/bridge.go#L71); [src/pkg/agent/tools/gateway.go:22](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/tools/gateway.go#L22); [src/pkg/gateway/handlers/config.go:341](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/gateway/handlers/config.go#L341)
- Commands run with the operator's full ambient credentials, and the product is designed to act on servers, databases and cloud platforms. (ASI03, T3; C1) — [src/pkg/agent/eino/localbackend/shell_windows.go:30](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/localbackend/shell_windows.go#L30); [README.md:41](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/README.md#L41)
- The approval gate is opt-in and can be disabled by the model; access control on the gateway through which approvals are answered is not locked down. (ASI09, ASI02, T10; C2) — [src/pkg/agent/runtime/runtime.go:190-194](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/runtime/runtime.go#L190-L194)
- No execution isolation: the shell runs on the host with the full environment. (ASI05, T11; C4) — [src/pkg/agent/eino/localbackend/shell_windows.go:30](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/localbackend/shell_windows.go#L30)
- A hijacked agent can exfiltrate and take irreversible infrastructure actions unattended. (ASI01, LLM01, T6; C5) — [src/pkg/agent/runtime/runtime.go:190-194](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/runtime/runtime.go#L190-L194)
- MCP servers launch as host subprocesses with the full environment, and the model can add new ones via config.patch. (ASI04, T17; C7) — [src/pkg/acp/mcp/client.go:37](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/acp/mcp/client.go#L37); [src/pkg/acp/mcp/client.go:53](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/acp/mcp/client.go#L53); [src/pkg/agent/tools/gateway.go:22](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/tools/gateway.go#L22)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

OpenOcta runs every command, MCP server and delegated CLI agent as the logged-in user with that user's full environment, so whatever cloud, Kubernetes, SSH or database credentials an ops engineer has on the machine are available to the model. There is no scoped identity and no authorization check between the model and those credentials. Worse, the agent is given a default tool (gateway_config) that can rewrite its own security configuration, so it can widen its own authority. For an ops agent whose purpose is to act on servers, networks, databases and cloud accounts, a hijacked agent holds the operator's full reach.

- **S L0:** Ambient OS-user authority: the shell and MCP subprocesses inherit the full process environment and nothing narrows credentials. — [src/pkg/agent/eino/localbackend/shell_windows.go:30](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/localbackend/shell_windows.go#L30); [src/pkg/acp/mcp/client.go:37](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/acp/mcp/client.go#L37); [src/pkg/agent/eino/localbackend/shell_unix.go:17](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/localbackend/shell_unix.go#L17) (verified)
  - *To reach the next level:* A dedicated identity or deterministic per-request authorization gate in front of credential use.
- **C L0:** No authorization layer exists on any tool path; built-in shell, MCP servers and local CLI agents all launch with ambient credentials. — [src/pkg/acp/mcp/client.go:53](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/acp/mcp/client.go#L53); [src/pkg/localagents/runner.go:89](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/localagents/runner.go#L89); searched `rg -n -i "seccomp|landlock|sandbox-exec|setpgid|chroot|firecracker|gvisor"` in `src` → 0 hits (verified)
  - *To reach the next level:* At least the main tool path would need an authorization check in code.
- **D L0:** The default desktop install runs with the user's full privileges; least privilege would require hardening that the product does not offer. — [src/pkg/config/config.go:119-155](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/config/config.go#L119-L155); [src/pkg/desktop/gateway.go:49](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/desktop/gateway.go#L49) (verified)
  - *To reach the next level:* A narrower default identity or credential set would be needed.
- **B L0:** The product's documented job is acting on monitoring, servers, databases and cloud platforms with the operator's credentials, so a hijack reaches the operator's whole infrastructure account. — [README.md:41](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/README.md#L41); [src/pkg/agent/eino/localbackend/shell_windows.go:30](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/localbackend/shell_windows.go#L30) (verified)
  - *To reach the next level:* Credentials would need to be scoped to one system or read-only.
- **Cap:** C1-SELFESC — The default gateway_config tool lets the model call config.patch, which rewrites ~/.openocta/openocta.json including the security section and MCP servers, so the agent can change its own permissions.

### C2 Approval gates — 0.23 (high)

An approval step exists but is off in the default configuration: it only activates when security.approvalQueue.enabled is explicitly true, and the desktop install writes a config without that section. When enabled it pauses only the shell 'execute' tool and shows the exact arguments, but file writes, MCP tools, the browser, the delegated local coding agents and the config-patching tool all run without approval. The model itself can switch the gate off through its config tool, and access control on the gateway through which approvals are answered is not locked down. Infrastructure actions such as deleting resources have no undo.

- **S L2:** When enabled, each execute call interrupts with the raw tool arguments and resumes with the stored arguments, but there are no risk tiers or argument-level policy in effect. — [src/pkg/agent/eino/approval_middleware.go:61-64](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/approval_middleware.go#L61-L64); [src/pkg/agent/eino/approval_middleware.go:69](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/approval_middleware.go#L69) (verified)
  - *To reach the next level:* Risk tiers deciding which calls need a human would be needed for L3.
- **C L1:** Only the tool named execute is gated; write_file, edit_file, MCP tools, browser, local_agent and gateway_config bypass the gate. — [src/pkg/agent/eino/engine.go:100-101](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/engine.go#L100-L101); [src/pkg/agent/tools/local_agent_tool.go:175](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/tools/local_agent_tool.go#L175); [src/pkg/agent/tools/bridge.go:71](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/tools/bridge.go#L71) (verified)
  - *To reach the next level:* Every built-in tool, including file writes, local_agent and gateway_config, would need to pass the gate.
- **D L0:** Approval is opt-in: the gate is enabled only when security.approvalQueue.enabled is explicitly true, and the default desktop config has no security section. — [src/pkg/agent/runtime/runtime.go:190-194](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/runtime/runtime.go#L190-L194); [src/pkg/config/config.go:119-155](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/config/config.go#L119-L155) (verified)
  - *To reach the next level:* The gate would need to be on by default.
- **B L0:** Wrongly approved or ungated actions are shell commands and delegated agents against production infrastructure with no checkpoint or undo. — [README.md:41](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/README.md#L41); [src/pkg/agent/eino/localbackend/shell_windows.go:30](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/localbackend/shell_windows.go#L30) (verified)
  - *To reach the next level:* Rollback or dry-run for common actions would be needed.
- **Cap:** C2-SELFAPPROVE — Non-principals can answer or disable approvals through the gateway.

### C3 Tool & action scoping — 0.00 (high)

The default tool set hands the model a raw shell, unrestricted file writes to any absolute path, a browser, delegation to locally installed coding agents, and a tool that rewrites the agent's own config. The only command filter is a configurable deny-list that is absent in the default config and whose matching is not a strict boundary. There are no argument allowlists or quantity bounds, so a misused tool can do anything the user can do on the machine and on every system it can reach.

- **S L0:** Raw passthrough: execute takes an arbitrary shell string, and write paths accept any absolute path with no containment. — [src/pkg/agent/eino/localbackend/shell_windows.go:30](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/localbackend/shell_windows.go#L30); [src/pkg/agent/eino/localbackend/paths.go:42-47](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/localbackend/paths.go#L42-L47) (verified)
  - *To reach the next level:* Typed validation such as resolved-path containment or command allowlists would be needed.
- **C L0:** No tool validates arguments against bounds; the deny-list validator is only built when a security section exists. — [src/pkg/agent/runtime/runtime.go:105-113](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/runtime/runtime.go#L105-L113) (verified)
  - *To reach the next level:* At least a few tools would need real argument validation.
- **D L0:** Everything is on by default: shell, file write, browser, local_agent and gateway_config. — [src/pkg/agent/runtime/runtime.go:367-371](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/runtime/runtime.go#L367-L371); [src/pkg/config/schema.go:100-104](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/config/schema.go#L100-L104); [src/pkg/agent/tools/bridge.go:71](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/tools/bridge.go#L71) (verified)
  - *To reach the next level:* Dangerous tools would need to be individually disableable at minimum, ideally off by default.
- **B L0:** General-purpose shell and delegated agents run against the whole machine and every reachable host. — [src/pkg/agent/eino/localbackend/shell_windows.go:30](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/localbackend/shell_windows.go#L30); [README.md:41](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/README.md#L41) (verified)
  - *To reach the next level:* Tools would need to be scoped to a workspace or project.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

Model-generated commands run directly on the host as the logged-in user (sh on macOS/Linux, cmd.exe on Windows) with the full environment. There is no isolation of any kind, and the sandbox-related options do not provide a complete boundary. MCP servers and delegated CLI agents are also launched on the host with the full environment. A malicious script or injected command reaches everything the user can.

- **S L0:** No isolation primitive: same-user subprocess on the host. — [src/pkg/agent/eino/localbackend/shell_windows.go:30](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/localbackend/shell_windows.go#L30) (verified)
  - *To reach the next level:* An OS-level sandbox (container, low-privilege user, Seatbelt/AppContainer) would be needed.
- **C L0:** No execution path is sandboxed: shell, MCP stdio servers and local CLI agents all run on the host. — [src/pkg/gateway/handlers/chat.go:2292](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/gateway/handlers/chat.go#L2292); [src/pkg/localagents/runner.go:89](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/localagents/runner.go#L89); searched `rg -n -i "seccomp|landlock|sandbox-exec|setpgid|chroot|firecracker|gvisor"` in `src` → 0 hits (verified)
  - *To reach the next level:* At least the main exec tool would need to be sandboxed.
- **D L0:** There is no sandbox to enable in the scored configuration. — [src/pkg/config/config.go:119-155](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/config/config.go#L119-L155) (verified)
  - *To reach the next level:* A sandbox would need to exist and be on by default.
- **B L0:** Host-equivalent reach: the shell runs as the user with home directory, ~/.ssh, cloud credentials and full network. — [src/pkg/agent/eino/localbackend/shell_windows.go:30](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/localbackend/shell_windows.go#L30); [src/pkg/acp/mcp/client.go:37](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/acp/mcp/client.go#L37) (verified)
  - *To reach the next level:* A workspace-only mount, scrubbed env and restricted network would be needed.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Web pages from the browser tool, MCP tool results, file contents, output from delegated agents and (when configured) messages from IM channels all enter the model's context the same way as the user's own instructions, with no provenance marking or approval tied to untrusted content. A hijacked agent has an unrestricted shell with network egress and the operator's credentials, so it can both exfiltrate data and take destructive actions without anyone approving. In addition, the local gateway's access control is not locked down.

- **S L0:** Nothing structurally limits a hijacked agent; there is no taint tracking, provenance, or approval triggered by untrusted content. — searched `rg -n -i "untrusted|provenance|prompt.injection|spotlight"` in `src/pkg` → 1 hits (only hit is a regex in the evolution-memory write filter (evolution/scan.go:13), not a provenance control); [src/pkg/agent/runtime/runtime.go:190-194](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/runtime/runtime.go#L190-L194) (verified)
  - *To reach the next level:* At least approval for some dangerous capabilities after untrusted content is read would be needed.
- **C L0:** Untrusted sources are not distinguished; tool results and browser content enter context like any other message, and IM channels have no sender allowlist. — searched `rg -n -i "untrusted|provenance|prompt.injection|spotlight"` in `src/pkg` → 1 hits (only hit is a regex in the evolution-memory write filter (evolution/scan.go:13), not a provenance control); searched `rg -n -i "allowfrom"` in `src/pkg/channels` → 0 hits (no sender allowlist enforcement in any IM channel adapter) (verified)
  - *To reach the next level:* At least one untrusted source would need to be handled.
- **D L0:** No defense exists to be on by default. — searched `rg -n -i "untrusted|provenance|prompt.injection|spotlight"` in `src/pkg` → 1 hits (only hit is a regex in the evolution-memory write filter (evolution/scan.go:13), not a provenance control); [src/pkg/config/config.go:119-155](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/config/config.go#L119-L155) (verified)
  - *To reach the next level:* A default-on control would be needed.
- **B L0:** Worst case: a hijacked agent can read secrets via the shell and send them anywhere, and run destructive infrastructure commands, all unattended. — [src/pkg/agent/eino/localbackend/shell_windows.go:30](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/localbackend/shell_windows.go#L30); [src/pkg/agent/runtime/runtime.go:190-194](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/runtime/runtime.go#L190-L194) (verified)
  - *To reach the next level:* Exfiltration or irreversible actions would need to require human approval.
- **Cap:** C5-PUBLICTRIGGER — Non-principals can send the agent instructions through the gateway while it holds the operator's credentials.

### C6 Memory, context & configuration integrity — 0.10 (high)

Text files in the agent workspace (.agents/evolution SOUL/PROMPT/MEMORY/USER.md) are loaded into the system prompt on every run, and the model can write them freely with its file and shell tools; a threat-pattern filter exists only on an internal API the model does not use. The model can also rewrite the main config through gateway_config, adding MCP servers that launch automatically or turning off security settings, with no confirmation. Memory is per agent workspace, not per sender, so on a shared IM channel one user's injected content persists for everyone. A single successful injection can therefore become a persistent backdoor that fires in later sessions.

- **S L0:** The model can write the evolution files that are re-injected into the system prompt, and can patch the config to add MCP servers or change security, with no gate. — [src/pkg/agent/runtime/runtime.go:180-186](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/runtime/runtime.go#L180-L186); [src/pkg/agent/evolution/store.go:295-305](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/evolution/store.go#L295-L305); [src/pkg/agent/evolution/store.go:152](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/evolution/store.go#L152); [src/pkg/agent/tools/gateway.go:22](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/tools/gateway.go#L22) (verified)
  - *To reach the next level:* Provenance on memory entries and a ban on project/agent-writable config changing security settings would be needed.
- **C L0:** No memory or config path is controlled: evolution files load unscanned and config.patch writes anything. — [src/pkg/agent/evolution/store.go:295-305](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/evolution/store.go#L295-L305); [src/pkg/gateway/handlers/config.go:341](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/gateway/handlers/config.go#L341) (verified)
  - *To reach the next level:* At least one store would need controlled writes.
- **D L1:** Memory is namespaced per agent workspace directory, but not per sender or session, and nothing in code stops the model writing other workspaces. — [src/pkg/agent/runtime/runtime.go:180-186](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/runtime/runtime.go#L180-L186); [src/pkg/agent/workspace.go:42](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/workspace.go#L42) (verified)
  - *To reach the next level:* Per-user/session namespaces enforced in code would be needed.
- **B L1:** Poisoned evolution text persists across all of the user's sessions and can drive tool use (it is part of the system prompt). — [src/pkg/agent/runtime/runtime.go:180-186](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/runtime/runtime.go#L180-L186) (verified)
  - *To reach the next level:* Poisoned context would need to be limited to text output or gated actions.
- **Cap:** none

### C7 Third-party extensions — 0.00 (high)

MCP servers are launched as host subprocesses with the agent's complete environment, the installer auto-enables a bundled MCP server on first run, and marketplace skills/MCPs are downloaded with no hash or signature check, through a download path that is not locked down. The model can add new MCP servers itself by patching the config through its gateway_config tool, and they launch on the next run without consent. A malicious or swapped extension inherits all of the user's access.

- **S L0:** No verification: marketplace downloads have no integrity check, and the model can choose and add MCP server commands via config.patch. — [src/pkg/gateway/http/site_install.go:85](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/gateway/http/site_install.go#L85); searched `rg -n -i "sha256|checksum|signature"` in `src/pkg/gateway/http/site_install.go src/pkg/acp` → 0 hits; [src/pkg/agent/tools/gateway.go:22](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/tools/gateway.go#L22) (verified)
  - *To reach the next level:* User-chosen sources with verified downloads and no model-chosen installs would be needed.
- **C L0:** No extension type is verified (MCP servers, skills, employees). — searched `rg -n -i "sha256|checksum|signature"` in `src/pkg/gateway/http/site_install.go src/pkg/acp` → 0 hits; [src/pkg/gateway/handlers/chat.go:2292](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/gateway/handlers/chat.go#L2292) (verified)
  - *To reach the next level:* At least one extension type would need verification.
- **D L0:** A bundled MCP server is enabled automatically on first run, and the model can add MCP servers silently through config.patch. — [src/pkg/bundled/install.go:35-37](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/bundled/install.go#L35-L37); [src/pkg/bundled/install.go:236-238](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/bundled/install.go#L236-L238); [src/pkg/agent/tools/bridge.go:71](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/tools/bridge.go#L71) (verified)
  - *To reach the next level:* Explicit user consent for every new extension would be needed.
- **B L0:** MCP servers run as the same user with the full process environment merged in. — [src/pkg/acp/mcp/client.go:37](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/acp/mcp/client.go#L37); [src/pkg/acp/mcp/client.go:53](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/acp/mcp/client.go#L53) (verified)
  - *To reach the next level:* A scrubbed environment per extension would be needed.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.00 (high)

With the default config, OpenOcta enables CozeLoop trace export, registering a global callback that ships agent traces (model inputs and outputs, tool calls) to a third-party service; its credential handling is not locked down. Model-provider keys sit in plaintext in openocta.json, the model can read the whole config through gateway_config, secret material is exposed through further output paths, and every subprocess inherits all secrets.

- **S L0:** Plaintext secrets in config, further secret-exposure paths, and config secrets readable by the model. — [src/pkg/gateway/handlers/config.go:123](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/gateway/handlers/config.go#L123) (verified)
  - *To reach the next level:* Secrets would need to come from env/keychain with at least one masked path.
- **C L0:** No path is protected. (verified)
  - *To reach the next level:* At least one path (logs) would need redaction.
- **D L0:** Trace export to a third party is on by default when the cozeloop section is absent, which is the default config. — [src/pkg/gateway/http/server.go:182](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/gateway/http/server.go#L182); [src/pkg/agent/eino/cozeloop.go:58-59](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/cozeloop.go#L58-L59); [src/pkg/config/config.go:119-155](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/config/config.go#L119-L155) (verified)
  - *To reach the next level:* Telemetry would need to be opt-in or at least content-free.
- **B L0:** Long-lived high-privilege credentials (cloud, kube, SSH, model keys) are reachable by the model and every subprocess. — [src/pkg/acp/mcp/client.go:37](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/acp/mcp/client.go#L37); [src/pkg/agent/eino/localbackend/shell_windows.go:30](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/localbackend/shell_windows.go#L30) (verified)
  - *To reach the next level:* Keys would need to be scoped and kept out of subprocess environments.
- **Cap:** none

### C9 Audit & traceability — 0.45 (high)

Each chat session is written to a JSONL transcript under ~/.openocta/agents/<id>/sessions that records tool calls and tool results with names and IDs. It does not record who approved what, there is no tamper protection, and the agent's own shell can edit or delete it. Tool results are buffered until the assistant turn is persisted, and write failures are only logged as warnings while the run continues.

- **S L2:** Structured per-session JSONL transcript with tool names, call IDs and results. — [src/pkg/session/paths.go:81](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/session/paths.go#L81); [src/pkg/session/transcript.go:37](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/session/transcript.go#L37) (verified)
  - *To reach the next level:* Actor attribution (approver, requesting principal) and correlation IDs would be needed.
- **C L2:** Tool calls flowing through the chat stream are recorded, but approvals/denials and config changes are not. — [src/pkg/gateway/handlers/chat.go:2574-2576](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/gateway/handlers/chat.go#L2574-L2576); searched `rg -n -i "approv"` in `src/pkg/session` → 0 hits (verified)
  - *To reach the next level:* Approvals and denials would need to be recorded.
- **D L2:** On by default and stored under the state directory rather than the workspace, but the agent's same-user shell can alter it. — [src/pkg/session/paths.go:81](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/session/paths.go#L81); [src/pkg/agent/eino/localbackend/shell_windows.go:30](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/localbackend/shell_windows.go#L30) (verified)
  - *To reach the next level:* The record would need to be written by a component the model cannot control.
- **B L1:** Best-effort: tool results are buffered until the turn is persisted and append failures are only warned. — [src/pkg/gateway/handlers/chat.go:2572-2580](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/gateway/handlers/chat.go#L2572-L2580) (verified)
  - *To reach the next level:* Errors would need to be surfaced and records flushed per action.
- **Cap:** none

### C10 Limits & kill switch — 0.45 (high)

Each run is capped at 50 model iterations and a 10-minute wall clock by default, and chat.abort cancels the run context, which stops the loop and kills the directly spawned shell process. There is no token or cost cap per run, and the timeout is read from config that the model can patch through gateway_config. Processes started in the background by shell commands or delegated agents are not tracked and can outlive a stop.

- **S L2:** Iteration cap (50) plus wall-clock run timeout enforced via context; no token/cost cap. — [src/pkg/agent/runtime/runtime.go:136](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/runtime/runtime.go#L136); [src/pkg/agent/eino/runner.go:107](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/runner.go#L107) (verified)
  - *To reach the next level:* A token or cost cap and rate limits on side-effecting tools would be needed.
- **C L2:** The run context covers the loop and its tool calls, but background processes and delegated agents' children are not counted or tracked. — [src/pkg/agent/eino/runner.go:107](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/runner.go#L107); [src/pkg/localagents/runner.go:89](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/localagents/runner.go#L89) (verified)
  - *To reach the next level:* Spawned processes and sub-agents would need to count against the same budget.
- **D L1:** Defaults are sensible, but agents.defaults.timeoutSeconds comes from config the model can patch via gateway_config. — [src/pkg/agent/runtime/timeouts.go:21-26](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/runtime/timeouts.go#L21-L26); [src/pkg/agent/tools/gateway.go:22](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/tools/gateway.go#L22) (verified)
  - *To reach the next level:* The model would need to be unable to raise its own limits.
- **B L2:** Moderate ceilings; abort cancels the run but orphaned background processes can continue. — [src/pkg/gateway/handlers/chat.go:3503-3504](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/gateway/handlers/chat.go#L3503-L3504); [src/pkg/agent/eino/runner.go:107](https://github.com/openocta/openocta/blob/0c7dac2284211facdfc813166fe648d093153bb6/src/pkg/agent/eino/runner.go#L107) (verified)
  - *To reach the next level:* Stop would need to cancel all pending work and leave no orphans.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: browser tool on by default (src/pkg/agent/runtime/runtime.go:371), MCP tool results (src/pkg/gateway/handlers/chat.go:2292), and a further untrusted input path · [B] sensitive data/systems: full process environment and operator credentials inherited by the shell (src/pkg/agent/eino/localbackend/shell_windows.go:30); config with model keys readable via gateway_config (src/pkg/gateway/handlers/config.go:123) · [C] state change / egress: unrestricted shell execute (src/pkg/agent/eino/engine.go:121-122) with approval off by default (src/pkg/agent/runtime/runtime.go:194) · Same default session? Yes

## Highest-impact improvements
1. Remove config.patch from the model-facing gateway_config tool (keep it read-only, without secrets), so the agent cannot change its own security settings or add MCP servers. — C6 S L0→L2, +0.150 before caps (Playbook 2)
2. Turn approval on by default and apply it to every mutating tool (write_file, edit_file, local_agent, MCP, gateway_config), using the existing deny/ask/allow policy as risk tiers. — C2 D L0→L3, +0.150 before caps (Playbook 5)
3. Make CozeLoop export opt-in and harden its credential handling and secret logging. — C8 D L0→L2, +0.100 before caps (Playbook 4)
4. Harden gateway access control. — C1 C L0→L1, +0.075 before caps (Playbook 4)
5. Run execute, MCP servers and local agents inside an OS sandbox (Seatbelt/AppContainer/container) with a scrubbed environment. — C4 S L0→L3, +0.225 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Upstream libraries were not in the clone: the behaviour of cloudwego/eino-ext's local backend on macOS/Linux (env inheritance, ValidateCommand hook) and of the CozeLoop callback handler (which span payloads it exports) is inferred from the OpenOcta call sites and library naming; the Windows shell path was read directly.
- The Control UI frontend (ui/) and IM channel adapters were reviewed only for the points cited; the Wails launcher and app-update code were not reviewed.
- The practical impact of some gateway behaviour depends on the browser and was not tested.
- The queue notes a license discrepancy (LICENSE is GPL-3.0 text while README badge says Apache-2.0); not relevant to the score.
- No reviewer-injection text was found in the repository.
