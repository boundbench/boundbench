# Defense-in-Depth Score: Linux MCP Server (RHEL Lightspeed)

**Repo:** https://github.com/rhel-lightspeed/linux-mcp-server · **Commit:** `11e303597c02d357144f9d05d4994e1d945f7e86` (1.7.0) · **Reviewed:** 2026-10-04
**What it is:** MCP server for read-only Linux system administration and diagnostics on RHEL-based systems, locally or over SSH.
**Category:** Infrastructure & Ops
**Scored configuration:** `linux-mcp-server` run as a local stdio server with no environment overrides: fixed read-only toolset, host_mode 'any' on a Linux host, no authorization policy, host-key verification on, and the operator's ambient SSH keys/agent.
**Agent surface (default):** code execution opt-in · filesystem write opt-in · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 5.6 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L3 | L4 | L2 | L2 | 0.72 | G1 | **0.50** (alt) | High |
| C2 | Approval gates | L2 | L2 | L3 | L4 | 0.65 | — | **0.65** | High |
| C3 | Tool & action scoping | L2 | L2 | L3 | L2 | 0.55 | — | **0.55** | High |
| C4 | Code-execution isolation | L2 | L0 | L1 | L0 | 0.20 | G2 | **0.20** | High |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L2 | L1 | L3 | L0 | 0.38 | — | **0.38** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | Medium |
| C10 | Limits & kill switch | L2 | L2 | L3 | L2 | 0.55 | — | **0.55** | High |

Controls where a risk surface exists: 3.62 / 8.0 (45%); 2 criteria scored SA (surface absent).

As shipped, Linux MCP Server only exposes fixed, genuinely read-only diagnostic commands, run as argument lists with timeouts and size caps, so it cannot change your systems. The main risk is what it can read and where it can reach: read_file opens any file your account can read, including SSH private keys, and the model can point any tool at any host name using your own SSH identity, which also gives a hijacked session a way to leak what it read. Its strong authorization policy and the sandbox for the optional script-execution mode are opt-in, and that script mode is much weaker than the default.

## Critical gaps
- In the opt-in run_script toolset the model's own readonly flag selects the sandbox profile, so modifying scripts run as root via sudo systemd-run with host networking, and the sandbox is not a complete boundary. (ASI05, T11; C4) — [src/linux_mcp_server/tools/run_script.py:189-202](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/run_script.py#L189-L202); [src/linux_mcp_server/tools/run_script.py:249-262](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/run_script.py#L249-L262)

## Criterion details

### C1 Identity & least privilege — 0.50 (high)

In its default local setup the server runs as whoever launched it and, with no policy file, the authorization middleware allows every tool call on every host. Remote calls use the operator's own SSH identity (default keys or agent) against any host name the model supplies, and local commands inherit the server's full environment. The project does ship a real, deterministic authorization layer: a YAML policy that maps tool, host and token claims to deny, local, or SSH with a specific per-rule key and user, and denies anything unmatched. It is opt-in, so it can only lift this criterion to the off-by-default ceiling.

- **default configuration** (default; raw 0.17 → 0.17)
  - **S L0:** With no policy configured, the server acts with the launching user's ambient authority: local commands as that user and SSH with asyncssh's default keys/agent for any model-chosen host. — [src/linux_mcp_server/server.py:266-270](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/server.py#L266-L270); [src/linux_mcp_server/connection/ssh.py:166-168](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/connection/ssh.py#L166-L168); [src/linux_mcp_server/utils/types.py:11-18](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/utils/types.py#L11-L18) (verified)
    - *To reach the next level:* Needs a narrowed default identity, e.g. a dedicated read-only SSH user or a default policy, rather than ambient keys.
  - **C L1:** Every tool passes through one middleware and execution context, but in the default stdio mode that context allows local and default-SSH execution for all tools, and local subprocesses inherit the full environment. — [src/linux_mcp_server/server.py:266-270](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/server.py#L266-L270); [src/linux_mcp_server/connection/ssh.py:488-494](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/connection/ssh.py#L488-L494) (verified)
    - *To reach the next level:* Default mode needs an authorization check per call and a scrubbed subprocess environment.
  - **D L1:** Default install runs with the operator's privilege; the read-only fixed toolset narrows what tools do, but not the credentials, and host_mode defaults to any host. — [src/linux_mcp_server/config.py:63-74](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/config.py#L63-L74); [src/linux_mcp_server/config.py:250-251](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/config.py#L250-L251) (verified)
    - *To reach the next level:* Needs a least-privilege default identity or default-deny policy without manual hardening.
  - **B L1:** The ambient SSH identity typically grants shell on every host in known_hosts, and read_file can return those private keys to the model, so a hijacked session exposes write-capable credentials across systems. — [src/linux_mcp_server/tools/storage.py:223-251](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/storage.py#L223-L251); [src/linux_mcp_server/utils/validation.py:49-68](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/utils/validation.py#L49-L68) (verified)
    - *To reach the next level:* Credentials should be scoped per host/read-only and not readable through the server's own tools.
- **opt-in YAML authorization policy (LINUX_MCP_POLICY_PATH)** (alt; raw 0.72, cap G1 → 0.50) ← counted
  - **S L3:** A deterministic policy maps each call's tool, host and token claims to deny/local/ssh_default/ssh_key, with a per-rule SSH key and user, failing closed when nothing matches. — [src/linux_mcp_server/auth_policy.py:147-159](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/auth_policy.py#L147-L159); [src/linux_mcp_server/auth_policy.py:197-198](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/auth_policy.py#L197-L198); [src/linux_mcp_server/server.py:313-320](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/server.py#L313-L320) (verified)
    - *To reach the next level:* Credentials are still static keys; no short-lived or downscoped per-request credentials.
  - **C L4:** All tool calls traverse the same middleware, which evaluates the policy against the requesting principal's claims; a missing policy file yields an empty rule set that denies everything. — [src/linux_mcp_server/server.py:284-302](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/server.py#L284-L302); [src/linux_mcp_server/auth_policy.py:205-207](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/auth_policy.py#L205-L207) (verified)
  - **D L2:** Operator sets the policy file path; widening is an operator edit to that file without warning. — [src/linux_mcp_server/config.py:271-272](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/config.py#L271-L272) (verified)
    - *To reach the next level:* Needs elevation that is time-bounded and a default that is minimal without operator setup.
  - **B L2:** With policy, the server still reaches any matched host with static SSH keys, but only via the fixed read-only tools unless run_script is enabled. — [src/linux_mcp_server/server.py:313-320](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/server.py#L313-L320) (verified)
    - *To reach the next level:* Keys are long-lived; blast radius is read access across many sensitive hosts.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.
- **Notes:** Default scored is stdio without policy. HTTP transport with auth providers (Google, GitHub, JWT, introspection) is also opt-in.

### C2 Approval gates — 0.65 (high)

As a tool server, the host owns the approval prompt, so this rates what the server tells the host. In the default 'fixed' toolset every tool is genuinely read-only and annotated readOnlyHint, the server refuses calls to tools outside the configured toolset, and nothing it exposes changes state, so a wrongly approved call can't change anything. The opt-in run_script toolset is weaker: the 'read-only' run_script tool trusts a flag the model sets (checked by an LLM gatekeeper), run_script_with_confirmation executes immediately on the server side; approval enforcement on the app-only script path is also not complete.

- **S L2:** Read and write tools are separate with accurate annotations in the default toolset, and the read-only default is server-enforced; but there is no server-side preview or confirmation for the opt-in mutating tools. — [src/linux_mcp_server/tools/storage.py:199-204](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/storage.py#L199-L204); [src/linux_mcp_server/tools/run_script.py:472-477](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/run_script.py#L472-L477); [src/linux_mcp_server/tools/run_script.py:63-65](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/run_script.py#L63-L65); [src/linux_mcp_server/tools/run_script.py:570-572](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/run_script.py#L570-L572) (verified)
  - *To reach the next level:* Needs a dry-run/preview or server-enforced confirmation for destructive script runs; run_script's readOnlyHint rests on a model-declared flag.
- **C L2:** Every default tool carries accurate annotations and unknown/out-of-toolset tools are rejected, but in the opt-in MCP-app mode approval enforcement does not cover every path. — [src/linux_mcp_server/server.py:259-262](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/server.py#L259-L262) (verified)
  - *To reach the next level:* Every tool path, including app-only tools, must carry risk annotations and server-checked approval.
- **D L3:** Read-only toolset is the default; enabling scripts needs LINUX_MCP_TOOLSET plus gatekeeper configuration, which the model cannot change. — [src/linux_mcp_server/config.py:250-251](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/config.py#L250-L251); [src/linux_mcp_server/config.py:310-318](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/config.py#L310-L318) (verified)
  - *To reach the next level:* Needs time-bounded elevated modes and approvals tied to an authenticated principal.
- **B L4:** In the default configuration no tool performs a consequential action; all fixed tools run fixed read-only argv commands. — [src/linux_mcp_server/config.py:250-251](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/config.py#L250-L251); [src/linux_mcp_server/tools/storage.py:199-204](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/storage.py#L199-L204); [src/linux_mcp_server/commands.py:90-95](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/commands.py#L90-L95) (verified)
- **Cap:** none
- **Notes:** One default tool's argument handling is not strict. No state-changing action is reachable through it.

### C3 Tool & action scoping — 0.55 (high)

Tools are narrow and well built in shape: each runs a fixed command as an argument list (no shell locally, shell-quoted over SSH), with typed parameters, numeric bounds, regex-validated PCP names, and a resolved-path allowlist for read_log_file. But path checking is shallow: read_file and the listing tools accept any absolute path without '..', so the model can read any file the user can read (SSH keys, cloud credentials), which makes the log allowlist moot. The host argument is any string with no allowlist, and other argument validation is not strict.

- **S L2:** Typed schemas, numeric bounds and some validation exist, but paths are only checked for absolute form, '..', leading '-' and control characters, with no containment or allowlist. — [src/linux_mcp_server/utils/validation.py:49-68](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/utils/validation.py#L49-L68); [src/linux_mcp_server/utils/validation.py:9](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/utils/validation.py#L9); [src/linux_mcp_server/tools/logs.py:118-124](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/logs.py#L118-L124) (verified)
  - *To reach the next level:* Needs allowlisted path roots with resolved-path containment, a host allowlist, and complete argument validation.
- **C L2:** Most built-in tools validate their arguments; the host argument does not, and other argument validation is not strict. — [src/linux_mcp_server/utils/types.py:11-18](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/utils/types.py#L11-L18); [src/linux_mcp_server/commands.py:440-443](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/commands.py#L440-L443) (verified)
  - *To reach the next level:* Needs one central validation layer covering every argument, including host.
- **D L3:** The default toolset is read-only; script execution requires operator configuration, and the model cannot enable tools. — [src/linux_mcp_server/config.py:250-251](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/config.py#L250-L251); [src/linux_mcp_server/config.py:310-318](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/config.py#L310-L318) (verified)
  - *To reach the next level:* Needs per-task tool allowlists.
- **B L2:** A misused tool reads anything the user can read on the local machine or any SSH host it can reach, but cannot write and outputs are size-bounded. — [src/linux_mcp_server/tools/storage.py:223-251](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/storage.py#L223-L251); [src/linux_mcp_server/config.py:238-239](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/config.py#L238-L239) (verified)
  - *To reach the next level:* Needs scoping to configured paths and hosts.
- **Cap:** none

### C4 Code-execution isolation — 0.20 (high)

The default read-only toolset never runs model-written code: every command is a fixed argument list. Code execution exists only in the opt-in run_script toolset, which is therefore what this criterion rates. There, scripts flagged read-only by the model run under sudo systemd-run with a read-only filesystem and no network, but scripts flagged as modifying run as root with only PrivateTmp and NoNewPrivileges; the wrapper is also not a complete boundary. Because the score reflects that opt-in path, it says little about the default install, which has no execution surface.

- **S L2:** For read-only scripts the systemd-run profile sets ReadOnlyPaths=/, RestrictAddressFamilies=AF_UNIX, PrivateTmp and NoNewPrivileges, but runs as root with no seccomp or capability drop. — [src/linux_mcp_server/tools/run_script.py:189-202](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/run_script.py#L189-L202); [src/linux_mcp_server/tools/run_script.py:249-262](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/run_script.py#L249-L262) (verified)
  - *To reach the next level:* Needs a non-root, capability-dropped, seccomp-restricted profile (or stronger) for every script.
- **C L0:** Scripts the model marks as modifying get no filesystem or network restriction, and the wrapper is not a complete boundary. — [src/linux_mcp_server/tools/run_script.py:249-262](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/run_script.py#L249-L262) (verified)
  - *To reach the next level:* Every script path must run inside the restricted profile.
- **D L1:** Execution is off by default, but once enabled the model chooses readonly=false to drop the profile, and server-side nothing waits for a human before run_script_with_confirmation executes. — [src/linux_mcp_server/config.py:250-251](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/config.py#L250-L251); [src/linux_mcp_server/tools/run_script.py:63-65](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/run_script.py#L63-L65); [src/linux_mcp_server/tools/run_script.py:570-572](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/run_script.py#L570-L572) (verified)
  - *To reach the next level:* Escalation out of the profile must need a server-enforced human approval.
- **B L0:** Modifying scripts run as root (via sudo) with host network and full filesystem. — [src/linux_mcp_server/tools/run_script.py:189-202](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/run_script.py#L189-L202) (verified)
  - *To reach the next level:* Needs a non-root, no-network, resource-limited, ephemeral environment.
- **Cap:** G2 — The model sets the readonly flag that selects the restricted profile, so it can opt a script out of the read-only/no-network sandbox at runtime.
- **Notes:** Score describes the opt-in run_script toolset; the default fixed toolset has no code-execution path.

### C5 Untrusted input blast radius — 0.30 (high)

The server feeds the model content that others can write: journal entries, log files, service output, process command lines and arbitrary file contents. Some tools return structured models (log entries with their unit or path), but read_file and several status tools return plain text with no marker that it is untrusted. In the default mode a hijacked model can read local secrets such as SSH keys and then leak them through the host name argument, which triggers DNS lookups and SSH connection attempts to any name it chooses. No tool can change state, so the worst case is data exfiltration, not destruction.

- **S L1:** Outputs are mostly plain text; some tools return structured models, but none carry an untrusted/provenance flag. — [src/linux_mcp_server/tools/storage.py:223-251](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/storage.py#L223-L251); [src/linux_mcp_server/tools/logs.py:171-176](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/logs.py#L171-L176) (verified)
  - *To reach the next level:* Needs structured outputs separating content from metadata on every tool, plus provenance.
- **C L1:** Only a few tools structure their results; file contents and status text come back unmarked. — [src/linux_mcp_server/tools/storage.py:223-251](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/storage.py#L223-L251); [src/linux_mcp_server/tools/services.py:87](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/services.py#L87) (verified)
  - *To reach the next level:* Needs consistent treatment across all content-returning tools.
- **D L2:** The structuring that exists is fixed in code and cannot be disabled by content. — [src/linux_mcp_server/tools/logs.py:171-176](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/logs.py#L171-L176) (verified)
  - *To reach the next level:* Needs explicit, warned control over provenance features (currently none exist to toggle).
- **B L1:** Default config combines untrusted input, readable secrets (any file via read_file) and an outbound channel (arbitrary SSH host names resolved and dialled), though no irreversible action is possible. — [src/linux_mcp_server/tools/storage.py:223-251](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/storage.py#L223-L251); [src/linux_mcp_server/utils/types.py:11-18](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/utils/types.py#L11-L18); [src/linux_mcp_server/connection/ssh.py:154-158](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/connection/ssh.py#L154-L158) (verified)
  - *To reach the next level:* Needs a host allowlist or no-egress mode so read secrets can't leave, or path restrictions on sensitive files.
- **Cap:** none

### C6 Memory, context & configuration integrity — 1.00 (high)

The server keeps no memory, conversation store or retrieval index, and reads no instruction or configuration files from a working directory. Settings come only from LINUX_MCP_* environment variables and command-line flags (pydantic-settings with no .env file), and the optional policy file comes from an explicit operator-supplied path. Validated scripts are held in memory only for the life of the process.

- **Structural absence:** searched `rg -n -S 'env_file|dotenv|load_dotenv|AGENTS.md|CLAUDE.md'` in `src/` → 0 hits (No .env or instruction-file loading.); searched `rg -n -S 'sqlite|vector|chroma|memory_store|pickle'` in `src/` → 0 hits (No persistence store.)

### C7 Third-party extensions — 1.00 (high)

The server loads no third-party code at runtime: no plugin system, no MCP client, no package installs on the model's behalf, and no model files. The MCP App HTML/JS is read from the package's own resources. A _vendor directory lets downstream packagers bundle dependencies at build time, which is ordinary supply chain rather than runtime extension loading.

- **Structural absence:** searched `rg -n -S 'entry_points|trust_remote_code|npx|torch.load|__import__|import_module'` in `src/` → 0 hits (No dynamic plugin or remote-code loading.)

### C8 Secrets & sensitive-data protection — 0.38 (high)

The few secrets the server holds (SSH key passphrase, OAuth client secrets) are SecretStr values, there is no telemetry, and tool-call logging redacts any parameter whose name looks sensitive. But nothing stops secrets reaching the model: read_file returns any file the user can read, including private SSH keys and cloud credential files, and local commands inherit the server's full environment, including gatekeeper API keys when that toolset is enabled.

- **S L2:** SecretStr for configured secrets and key-name redaction in tool-call logs. — [src/linux_mcp_server/config.py:242-244](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/config.py#L242-L244); [src/linux_mcp_server/audit.py:24-37](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/audit.py#L24-L37); [src/linux_mcp_server/audit.py:136](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/audit.py#L136) (verified)
  - *To reach the next level:* Needs redaction or blocking of secrets before model-bound output, and a secret manager for stored credentials.
- **C L1:** Only the tool-call log path is protected; model-bound tool results and subprocess environments are not. — [src/linux_mcp_server/tools/storage.py:223-251](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/storage.py#L223-L251); [src/linux_mcp_server/connection/ssh.py:488-494](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/connection/ssh.py#L488-L494); [src/linux_mcp_server/gatekeeper/openai_client.py:87](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/gatekeeper/openai_client.py#L87) (verified)
  - *To reach the next level:* Needs protection on model-bound results and subprocess environments too.
- **D L3:** No telemetry exists, logging defaults to INFO, and log redaction is always on. — [src/linux_mcp_server/config.py:232](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/config.py#L232); [src/linux_mcp_server/audit.py:136](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/audit.py#L136) (verified)
  - *To reach the next level:* Needs minimised/encrypted stored records by default.
- **B L0:** Long-lived, high-privilege SSH private keys and other credential files are readable by the model through read_file. — [src/linux_mcp_server/tools/storage.py:223-251](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/storage.py#L223-L251); [src/linux_mcp_server/utils/validation.py:49-68](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/utils/validation.py#L49-L68) (verified)
  - *To reach the next level:* Needs sensitive paths blocked or an ambient-only design that never exposes key material.
- **Cap:** none

### C9 Audit & traceability — 0.50 (medium)

Every one of the server's 30 tools is wrapped by a logging decorator that writes a structured record (tool, sanitized arguments, host, status, duration, timestamp) to rotating text and JSON files under ~/.local/share, and remote commands are logged with their exact command line and exit code. Local commands are logged only at debug level, and in the default local mode there is no user identity and authorization decisions are logged only at debug. The files are ordinary user-writable files kept for ten days.

- **S L2:** Structured per-call records with arguments, status and timestamps in JSON and text logs. — [src/linux_mcp_server/audit.py:136-158](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/audit.py#L136-L158); [src/linux_mcp_server/logging_config.py:133-158](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/logging_config.py#L133-L158) (verified)
  - *To reach the next level:* Needs actor attribution (requesting principal, approver) in the default mode and tamper evidence.
- **C L2:** All built-in tools are logged and remote commands are recorded, but local commands and authorization decisions are only at debug level in stdio mode. — [src/linux_mcp_server/connection/ssh.py:411](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/connection/ssh.py#L411); [src/linux_mcp_server/connection/ssh.py:273](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/connection/ssh.py#L273); [src/linux_mcp_server/server.py:272-282](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/server.py#L272-L282) (verified)
  - *To reach the next level:* Needs every executed command and every approval/denial recorded at the default level.
- **D L2:** On by default and stored outside any workspace, but in a file the server's own user (and its opt-in script tools) can alter. — [src/linux_mcp_server/config.py:231-233](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/config.py#L231-L233); [src/linux_mcp_server/logging_config.py:133-158](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/logging_config.py#L133-L158) (verified)
  - *To reach the next level:* Needs the log written by a component the model can't control.
- **B L2:** Records are written per call via stdlib file handlers that flush each record; logging failures don't block actions. — [src/linux_mcp_server/logging_config.py:133-158](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/logging_config.py#L133-L158) (inferred)
  - *To reach the next level:* Needs replayable, durable records and fail-closed behaviour for high-risk actions.
- **Cap:** none

### C10 Limits & kill switch — 0.55 (high)

Every command the server runs, locally or over SSH, has a 30-second default timeout; file reads are capped at 1 MiB, log and journal reads at 10,000 lines, and listings at depth one. There are no rate or concurrency limits, some outputs (process list, service list) are unbounded, and a local timeout kills only the direct child process, not any processes it spawned.

- **S L2:** Server-enforced timeouts and output caps on most operations. — [src/linux_mcp_server/config.py:259-260](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/config.py#L259-L260); [src/linux_mcp_server/config.py:238-239](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/config.py#L238-L239); [src/linux_mcp_server/tools/logs.py:118-124](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/tools/logs.py#L118-L124); [src/linux_mcp_server/commands.py:164](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/commands.py#L164) (verified)
  - *To reach the next level:* Needs caps on every output plus concurrency or rate limits.
- **C L2:** The timeout applies to every local and remote command. — [src/linux_mcp_server/connection/ssh.py:496-499](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/connection/ssh.py#L496-L499); [src/linux_mcp_server/connection/ssh.py:248](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/connection/ssh.py#L248) (verified)
  - *To reach the next level:* Needs process-group kill so spawned processes count against the limit.
- **D L3:** Sensible defaults that only the operator can change; the model has no way to raise them. — [src/linux_mcp_server/config.py:259-260](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/config.py#L259-L260) (verified)
  - *To reach the next level:* Needs hard ceilings configuration can't exceed.
- **B L2:** Moderate ceilings; a timed-out local command's child processes may continue. — [src/linux_mcp_server/connection/ssh.py:496-499](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/connection/ssh.py#L496-L499); [src/linux_mcp_server/connection/ssh.py:488-494](https://github.com/rhel-lightspeed/linux-mcp-server/blob/11e303597c02d357144f9d05d4994e1d945f7e86/src/linux_mcp_server/connection/ssh.py#L488-L494) (verified)
  - *To reach the next level:* Needs cancellation of in-flight work and no orphaned processes.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: journal, log files, arbitrary files and process/service text returned to the model (src/linux_mcp_server/tools/storage.py:226-251) · [B] sensitive data/systems: any file readable by the server user, including ~/.ssh private keys (src/linux_mcp_server/utils/validation.py:49-68) · [C] state change / egress: model-chosen SSH host names resolved and dialled with ambient credentials (src/linux_mcp_server/utils/types.py:11-18); no state change in default toolset · Same default session? Yes

## Highest-impact improvements
1. Restrict read_file/list tools to configurable allowed path roots with resolved-path containment and a default denylist for ~/.ssh, credential files and /etc/shadow. — C8 B L0→L2, +0.100 before caps (Playbook 4)
2. Add a host allowlist (or default to known_hosts entries only, checked before DNS resolution) for the host argument. — C5 B L1→L3, +0.100 before caps (Playbook 1)
3. Validate every tool argument against a strict pattern before it reaches a command line. — C3 S L2→L3, +0.075 before caps (Playbook 3)
4. Run every script (not only model-flagged read-only ones) under a non-root, no-network systemd-run profile. — C4 C L0→L3, +0.225 before caps (Playbook 3)
5. Log local command lines and authorization decisions at INFO in stdio mode. — C9 C L2→L3, +0.075 before caps

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The project's installation docs live on an external site and were not reviewed; the scored default is taken from the code's config defaults (stdio, fixed toolset).
- Behaviour of asyncssh (default key/agent use, DNS resolution before host-key check) and of system tools is inferred from library/tool documentation, not observed.
- The LLM gatekeeper prompts and eval suite for the opt-in run_script mode were skimmed, not audited in depth.
