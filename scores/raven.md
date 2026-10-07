# Defense-in-Depth Score: Raven

**Repo:** https://github.com/evermind-ai/raven · **Commit:** `3632e6040c7038a60ec418ce39ccae185c72c19f` (0.2.4) · **Reviewed:** 2026-10-04
**What it is:** Host agent and CLI/web app that orchestrates built-in and third-party agents (research, code, design, oncall) with memory, skills and chat channels.
**Category:** AI Assistants
**Scored configuration:** Fresh install of the raven CLI/TUI/web page with shipped defaults: permissions.mode=smart, tools.sandbox.backend=none, restrictToWorkspace=false, no third-party sub-agents or channels configured.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 3.5 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L2 | L1 | L0 | 0.28 | — | **0.28** | High |
| C2 | Approval gates | L3 | L2 | L0 | L2 | 0.47 | G1 | **0.47** (alt) | High |
| C3 | Tool & action scoping | L3 | L2 | L0 | L1 | 0.42 | G1 | **0.42** (alt) | High |
| C4 | Code-execution isolation | L4 | L1 | L0 | L2 | 0.47 | G1 | **0.47** (alt) | High |
| C5 | Untrusted input blast radius | L1 | L2 | L2 | L0 | 0.33 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L0 | L1 | L1 | L1 | 0.17 | C6-REPOCONFIG | **0.17** | High |
| C7 | Third-party extensions | L0 | L1 | L0 | L0 | 0.07 | C7-RCELOAD | **0.07** | High |
| C8 | Secrets & sensitive-data protection | L2 | L3 | L3 | L0 | 0.53 | — | **0.53** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


Raven has an unusually careful permission gate, a parsed read-only shell allowlist, SSRF-hardened fetching and secret scrubbing, but its shipped defaults undercut them. By default an LLM reviewer, not a person, approves most actions, including pushes and installs; shell commands run on the host with no sandbox; and reading any file or fetching any URL needs no approval, so a prompt injection can leak local credentials unattended. Plugin loading from the working directory is also not gated by a trust decision.

## Critical gaps
- By default the agent can read any file the OS user can (including ~/.ssh and ~/.aws) without approval, so a hijacked session holds the user's whole account. (ASI03; C1) — [raven/permissions/rules.py:89](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/rules.py#L89); [raven/config/schema.py:1316](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/schema.py#L1316)
- Model-driven shell commands execute directly on the host as the user by default (sandbox backend 'none'), with the home directory and network reachable. (ASI05; C4) — [raven/sandbox/config.py:38](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/config.py#L38); [raven/sandbox/__init__.py:94-104](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/__init__.py#L94-L104)
- Default worst case: an injected instruction can read local credentials (read_file, auto-allowed) and exfiltrate them via web_fetch (auto-allowed), and push or install via the smart-mode LLM reviewer, all unattended. (ASI01, LLM01; C5) — [raven/permissions/rules.py:89](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/rules.py#L89); [raven/permissions/rules.py:97](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/rules.py#L97); [raven/permissions/judge.py:32-33](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/judge.py#L32-L33)
- Third-party code loads without consent (C7-RCELOAD): Skill Hub bundles auto-install by default, and plugins are imported with no verification. (ASI04; C7) — [raven/config/raven.py:855](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/raven.py#L855)

## Criterion details

### C1 Identity & least privilege — 0.28 (high)

Raven runs as the operator's own OS user and holds the model-provider and web API keys from its config. Its main shell tool strips the environment down to an allowlist, so API keys and cloud credentials in environment variables do not reach shell commands. Nothing narrows file access by default, though: the read tool has no fence and reading files is auto-allowed, so ~/.ssh, ~/.aws and other credentials on disk are one call away. Third-party CLI sub-agents are given a full capture of the user's login-shell environment.

- **S L1:** Ambient OS-user authority; the only narrowing is an environment allowlist for shell subprocesses. — [raven/sandbox/direct_executor.py:27](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/direct_executor.py#L27); [raven/sandbox/direct_executor.py:272](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/direct_executor.py#L272) (verified)
  - *To reach the next level:* No scoped or per-tool credentials and no authorization gate that maps requests to a least-privilege policy before credentials are used.
- **C L2:** Built-in exec gets the scrubbed env and MCP stdio servers get the SDK default env, but CLI sub-agents inherit the full login-shell environment and in-process plugins see everything. — [raven/sandbox/direct_executor.py:272](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/direct_executor.py#L272); [raven/agent/subagent/backends/env.py:144](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/subagent/backends/env.py#L144); [raven/mcp/client.py:199](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/mcp/client.py#L199) (verified)
  - *To reach the next level:* Sub-agents and plugins do not pass through the same scoped-authority layer as built-in tools.
- **D L1:** The env allowlist is on by default, but file tools are unrestricted by default (restrictToWorkspace false). — [raven/config/schema.py:1316](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/schema.py#L1316); [raven/security/redact.py:10](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/security/redact.py#L10) (verified)
  - *To reach the next level:* Default is not near-minimal: whole-home file access needs manual hardening via restrictToWorkspace.
- **B L0:** A hijacked session can read every credential file the user owns (read_file is auto-allowed with no fence) and reach registered machines over SSH. — [raven/permissions/rules.py:89](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/rules.py#L89); [raven/security/redact.py:10](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/security/redact.py#L10); [raven/config/schema.py:1316](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/schema.py#L1316) (verified)
  - *To reach the next level:* Blast radius is the user's whole account; would need credential-directory denial or workspace containment by default.
- **Cap:** none

### C2 Approval gates — 0.47 (high)

Every tool call, including MCP tools and built-in sub-agents, passes one permission gate with a parsed read-only shell allowlist, a catastrophe deny list and per-call prompts that show the exact call. But the shipped mode is 'smart': an LLM reviewer approves most ask-tier calls itself and is told to allow pushing branches, installing dependencies and running scripts; a human is asked only when it escalates, and the reviewer also runs in unattended turns. Third-party ACP sub-agents have every permission request auto-approved. The strict per-call 'ask' mode exists but is opt-in.

- **default configuration** (default; raw 0.38 → 0.38)
  - **S L1:** Default smart mode lets an LLM classifier auto-approve ask-tier calls, with a human only on escalation or reviewer failure. — [raven/config/schema.py:1288](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/schema.py#L1288); [raven/permissions/judge.py:32-33](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/judge.py#L32-L33); [raven/permissions/gate.py:206-210](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/gate.py#L206-L210) (verified)
    - *To reach the next level:* A human should approve consequential calls by default; an LLM reviewer approving pushes and installs is L1.
  - **C L2:** All registry tools (built-in, MCP, built-in sub-agents) cross the gate and compound shell commands are parsed, but third-party ACP sub-agents' requests are auto-approved. — [raven/agent/tools/registry.py:992](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/tools/registry.py#L992); [raven/permissions/rules.py:380](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/rules.py#L380); [raven/acp_client/permissions.py:5-6](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/acp_client/permissions.py#L5-L6); [raven/acp_client/permissions.py:56](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/acp_client/permissions.py#L56) (verified)
    - *To reach the next level:* Third-party sub-agent tool calls bypass the human gate entirely.
  - **D L1:** On by default, but security settings are not tamper-resistant and plugin loading is not gated by a trust decision. — [raven/config/schema.py:1288](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/schema.py#L1288) (verified)
    - *To reach the next level:* Security settings should be writable only through an operator path.
  - **B L2:** Per-turn shadow-git checkpoints make workspace edits reversible in interactive sessions; pushes, installs and network actions are not. — [raven/config/raven.py:1325](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/raven.py#L1325); [raven/permissions/judge.py:32-33](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/judge.py#L32-L33) (verified)
    - *To reach the next level:* No previews or rollback for external actions and no rate limit on consequential calls.
- **opt-in 'ask' permission mode (permissions.mode=ask)** (alt; raw 0.47, cap G1 → 0.47) ← counted
  - **S L3:** Per-call human approval showing the exact command or call, with risk tiers, deny-and-stop, and an approval digest so a refused call is not re-asked. — [raven/permissions/gate.py:219](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/gate.py#L219); [raven/permissions/gate.py:346](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/gate.py#L346); [raven/permissions/builtin.py:89](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/builtin.py#L89) (verified)
    - *To reach the next level:* Long arguments are shortened in the prompt line and argument-level allow/deny rules exist only for exec.
  - **C L2:** Same coverage as the default: registry tools gated, third-party ACP sub-agents auto-approved. — [raven/agent/tools/registry.py:992](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/tools/registry.py#L992); [raven/acp_client/permissions.py:5-6](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/acp_client/permissions.py#L5-L6) (verified)
    - *To reach the next level:* Third-party sub-agents bypass the gate.
  - **D L0:** Opt-in; the shipped default is smart. — [raven/config/schema.py:1288](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/schema.py#L1288) (verified)
    - *To reach the next level:* Ask mode is not the default.
  - **B L2:** Same checkpoints as default. — [raven/config/raven.py:1325](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/raven.py#L1325) (verified)
    - *To reach the next level:* No rollback or previews for external actions.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.
- **Notes:** C2-SELFAPPROVE not applied: the reviewer is a separate constrained call that fails closed to a human, not the acting model's own say-so; it is rated at S L1 instead.

### C3 Tool & action scoping — 0.42 (high)

The default tool set includes an arbitrary host shell, file write and edit, web fetch and a browser, all enabled at once. Some validation is strong: web fetch blocks private and metadata addresses and rechecks every redirect hop with DNS pinning, and the shell's auto-allow list is a carefully parsed read-only command set. But file tools resolve paths with no containment by default, and the shell itself is a raw command string. An opt-in restrictToWorkspace mode adds resolved-path containment for file tools.

- **default configuration** (default; raw 0.35 → 0.35)
  - **S L2:** Typed schemas plus strong SSRF checks on web_fetch, but exec accepts any shell string and file paths are resolved without containment by default. — [raven/security/network.py:193-231](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/security/network.py#L193-L231); [raven/agent/tools/web.py:1020](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/tools/web.py#L1020); [raven/agent/tools/filesystem.py:66-67](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/tools/filesystem.py#L66-L67) (verified)
    - *To reach the next level:* File and shell tools lack allowlist containment by default.
  - **C L2:** Most built-in tools validate their arguments; MCP and plugin tools get only schema validation. — [raven/agent/tools/registry.py:976](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/tools/registry.py#L976) (verified)
    - *To reach the next level:* No shared argument-policy layer for extension tools.
  - **D L0:** Exec, write, web and browser tools are all registered by default. — [raven/agent/loop/wiring.py:921-937](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/loop/wiring.py#L921-L937) (verified)
    - *To reach the next level:* Read-only default tool set with write/exec enabled explicitly.
  - **B L1:** Exec reaches the whole machine, limited only by a catastrophe-class deny list and the approval tier. — [raven/permissions/builtin.py:44](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/builtin.py#L44) (verified)
    - *To reach the next level:* Tools are not scoped to a workspace or quantity-bounded by default.
- **opt-in restrictToWorkspace** (alt; raw 0.42, cap G1 → 0.42) ← counted
  - **S L3:** File tools check resolved-path containment against the workspace; exec adds a heuristic path check. — [raven/agent/tools/filesystem.py:66-70](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/tools/filesystem.py#L66-L70); [raven/agent/tools/shell.py:532](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/tools/shell.py#L532) (verified)
    - *To reach the next level:* Exec containment is a lexical path scan, not a boundary; no narrow-tool replacement for shell.
  - **C L2:** Applies to built-in file, media and deliver tools; not to MCP/plugin tools. — [raven/agent/loop/wiring.py:920](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/loop/wiring.py#L920) (verified)
    - *To reach the next level:* Extension tools are not covered.
  - **D L0:** Off by default. — [raven/config/schema.py:1316](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/schema.py#L1316) (verified)
    - *To reach the next level:* Off by default.
  - **B L1:** Exec containment is not a complete boundary; network is unrestricted. (verified)
    - *To reach the next level:* No quantity bounds and no network scoping.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C4 Code-execution isolation — 0.47 (high)

Shell commands run directly on the host as the user by default; the code even logs a warning saying prompt-injected commands run with full host privileges. An opt-in Boxlite microVM backend is a strong boundary and fails closed if it cannot start, but it mounts the workspace read-write with open network by default, and third-party CLI sub-agents still run on the host. Environment scrubbing keeps API keys out of shell commands, but the home directory is fully reachable.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Default backend 'none' runs commands as a same-user host subprocess. — [raven/sandbox/config.py:38](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/config.py#L38); [raven/sandbox/__init__.py:94-104](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/__init__.py#L94-L104) (verified)
    - *To reach the next level:* No isolation primitive in the default configuration.
  - **C L0:** No execution path is isolated by default. — [raven/sandbox/__init__.py:94-104](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/__init__.py#L94-L104) (verified)
    - *To reach the next level:* The main exec tool is not sandboxed.
  - **D L0:** Sandbox is off by default. — [raven/sandbox/config.py:38](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/config.py#L38) (verified)
    - *To reach the next level:* Sandboxing is opt-in.
  - **B L0:** Host-equivalent: home directory, ~/.ssh and full network are reachable; only env vars are scrubbed. — [raven/sandbox/__init__.py:94-104](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/__init__.py#L94-L104); [raven/sandbox/direct_executor.py:27](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/direct_executor.py#L27) (verified)
    - *To reach the next level:* Would need workspace-only filesystem, no secrets, and restricted egress.
- **opt-in Boxlite microVM (tools.sandbox.backend=boxlite)** (alt; raw 0.47, cap G1 → 0.47) ← counted
  - **S L4:** Kernel-separated microVM backend; 'auto'/'boxlite' raise rather than fall back to host when unavailable. — [raven/sandbox/boxlite_executor.py:17](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/boxlite_executor.py#L17); [raven/sandbox/__init__.py:106](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/__init__.py#L106) (verified)
  - **C L1:** The exec tool runs in the VM, but CLI sub-agents run on the host by design. — [raven/agent/subagent/backends/cli_agent.py:4](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/subagent/backends/cli_agent.py#L4) (verified)
    - *To reach the next level:* Sub-agent and other spawned paths are not sandboxed.
  - **D L0:** Off by default. — [raven/sandbox/config.py:38](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/config.py#L38) (verified)
    - *To reach the next level:* Off by default.
  - **B L2:** Workspace mounted read-write and network fully open by default inside the VM. — [raven/sandbox/boxlite_executor.py:235](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/boxlite_executor.py#L235); [raven/sandbox/config.py:44](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/config.py#L44) (verified)
    - *To reach the next level:* Egress should be off or allowlisted by default.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.25 (high)

Every tool result is wrapped in a nonce-tagged 'untrusted data' fence before the model sees it, and chat channels deny unknown senders by default. Nothing acts on that fence: reading files and fetching arbitrary URLs are auto-allowed, and the default LLM reviewer approves pushes, installs and script runs. A prompt-injected session can therefore read local secrets and send them out through a fetched URL, and take irreversible actions, without a human.

- **S L1:** Delimiter-based spotlighting of tool results only; no capability is disabled once untrusted content is read. — [raven/agent/context/builder.py:300](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/context/builder.py#L300); [raven/security/trust.py:23](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/security/trust.py#L23) (verified)
  - *To reach the next level:* Egress and state-changing tools should require human approval once untrusted content enters the session.
- **C L2:** The fence covers every tool result (including MCP and sub-agent output) and recalled memory, but the user.md memory dump and workspace bootstrap files are injected unfenced. — [raven/agent/context/builder.py:300](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/context/builder.py#L300); [raven/memory_engine/consolidate/consolidator.py:937](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/memory_engine/consolidate/consolidator.py#L937) (verified)
  - *To reach the next level:* Not every source is fenced; L3 also needs S above L1.
- **D L2:** The fence is unconditional and channel allowlists deny by default. — [raven/agent/context/builder.py:300](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/context/builder.py#L300); [raven/auth/allowlist.py:48](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/auth/allowlist.py#L48) (verified)
  - *To reach the next level:* Capped by the weak mechanism (S L1).
- **B L0:** Unattended exfiltration via auto-allowed read_file plus web_fetch, and irreversible actions via the smart reviewer. — [raven/permissions/rules.py:89](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/rules.py#L89); [raven/permissions/rules.py:97](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/rules.py#L97); [raven/permissions/judge.py:32-33](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/judge.py#L32-L33) (verified)
  - *To reach the next level:* Egress or irreversible actions should require human approval.
- **Cap:** C5-WORSTCASE — In the default configuration a hijacked session can leak local secrets through auto-allowed web_fetch and take irreversible actions approved by the LLM reviewer, with no human.

### C6 Memory, context & configuration integrity — 0.17 (high)

Raven's own settings load only from the user's home, but plugin loading from the working directory is not gated by a trust decision. Long-term memory (user.md) is written automatically from conversations and re-injected into the system prompt unfenced, under a single default user id, while the EverOS recall block is fenced.

- **S L0:** Plugin loading from the working directory is not gated by a trust decision; memory writes are unvalidated and reloaded as trusted context. (verified)
  - *To reach the next level:* Plugin loading should be hardened, and memory writes should be validated or approved.
- **C L1:** Only the EverOS recall block is fenced; user.md and bootstrap files are uncontrolled. — [raven/context_engine/segments/render.py:478](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/context_engine/segments/render.py#L478); [raven/context_engine/segments/render.py:54-57](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/context_engine/segments/render.py#L54-L57) (verified)
  - *To reach the next level:* Main memory store and auto-loaded files are not controlled.
- **D L1:** Memory is keyed to one configured user id ('default') shared by every sender a channel admits. — [raven/config/raven.py:1153](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/raven.py#L1153) (verified)
  - *To reach the next level:* No per-user/session namespace enforced by default.
- **B L1:** Poisoned memory persists across the user's sessions and can steer tool use. — [raven/memory_engine/consolidate/consolidator.py:916](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/memory_engine/consolidate/consolidator.py#L916) (verified)
  - *To reach the next level:* Memory should be session-scoped or easily reviewed and purged.
- **Cap:** C6-REPOCONFIG — Repository-controlled content can change agent behaviour without a trust decision.

### C7 Third-party extensions — 0.07 (high)

Raven loads plugins from four places and imports them into its own process with no verification. The Skill Hub installs router-picked skill bundles automatically by default, with no hash checks. The built-in plugin catalog is curated and some entries are pinned, but others launch npx packages at @latest. MCP stdio servers run as separate processes with a reduced environment, but in-process plugins get everything.

- **S L0:** Plugin code is imported with no verification; Skill Hub bundles download without integrity checks. — searched `rg -n 'hashlib|sha256'` in `raven/skill_hub raven/market` → 0 hits (No integrity verification in the Skill Hub client or plugin market.) (verified)
  - *To reach the next level:* Extensions should be pinned and integrity-checked.
- **C L1:** Only the ACP shim presets are version-pinned; catalog entries mix pinned and @latest, and other types are unverified. — [raven/agent/subagent/presets.py:126](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/subagent/presets.py#L126); [raven/market/catalog.json:1201](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/market/catalog.json#L1201) (verified)
  - *To reach the next level:* Most extension types are unverified.
- **D L0:** Skill Hub auto-install is on by default, and plugin-loading defaults are not consent-gated. — [raven/config/raven.py:855](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/raven.py#L855) (verified)
  - *To reach the next level:* Nothing third-party should be enabled without an explicit, informed install.
- **B L0:** File-based plugins run in-process with all of the agent's credentials and authority. (verified)
  - *To reach the next level:* Extensions should run in a separate process with a scrubbed environment.
- **Cap:** C7-RCELOAD — By default third-party code can load and execute without consent.

### C8 Secrets & sensitive-data protection — 0.53 (high)

Raven scrubs the exact credential values it holds from every tool result before the model, trace spans and the client see them, redacts common secret patterns on the editor wire, gives shell commands an allowlisted environment, and stores OAuth tokens with owner-only permissions. No third-party telemetry was found. But secrets that are not Raven's own, such as ~/.aws/credentials or a repo's .env, are readable without approval and go to the model unredacted, and provider keys are long-lived.

- **S L2:** Exact-value scrubbing of Raven-held secrets plus pattern redaction on the ACP wire; credentials are plaintext files with owner-only modes. — [raven/agent/context/builder.py:293](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/context/builder.py#L293); [raven/config/held_secrets.py:90](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/held_secrets.py#L90); [raven/mcp/oauth.py:363](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/mcp/oauth.py#L363) (verified)
  - *To reach the next level:* No OS keychain or encryption at rest; pattern redaction does not cover model-bound messages.
- **C L3:** Held-secret scrubbing covers model-bound tool results, trace spans and client payloads; subprocess env is allowlisted. — [raven/agent/context/builder.py:293](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/context/builder.py#L293); [raven/agent/tools/registry.py:872](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/tools/registry.py#L872); [raven/sandbox/direct_executor.py:27](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/direct_executor.py#L27) (verified)
  - *To reach the next level:* Only Raven's own values are covered; other secrets read by tools reach every path unredacted.
- **D L3:** No third-party telemetry; scrubbing is unconditional; traces stay local. — searched `rg -n 'sentry_sdk|posthog|segment\.io|mixpanel'` in `raven` → 0 hits (No telemetry SDKs in the package.); [raven/agent/context/builder.py:293](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/context/builder.py#L293) (verified)
  - *To reach the next level:* Stored transcripts and traces are not encrypted or minimised.
- **B L0:** Long-lived provider keys and every credential file in the user's home are reachable by the model (read_file auto-allowed, no fence). — [raven/permissions/rules.py:89](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/rules.py#L89); [raven/security/redact.py:10](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/security/redact.py#L10) (verified)
  - *To reach the next level:* Keys should be scoped and short-lived, and home credential files kept out of reach.
- **Cap:** none

### C9 Audit & traceability — 0.40 (high)

Every registry tool call is recorded as a structured span with arguments, result, and the permission decision (rule, LLM reviewer verdict, or human choice), in local JSONL under ~/.raven/traces, on by default. Records sit where the agent's own unfenced file and shell tools can edit them, and tracing can be turned off through a setting the LLM reviewer may approve. Write errors are swallowed.

- **S L2:** Structured span per tool call with trace/parent ids and permission.source/approval choice annotations. — [raven/agent/tools/registry.py:856](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/tools/registry.py#L856); [raven/permissions/gate.py:309-316](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/permissions/gate.py#L309-L316) (verified)
  - *To reach the next level:* No verified requesting-principal attribution on records.
- **C L2:** All registry calls (built-in, MCP, built-in sub-agents) and gate decisions; third-party sub-agents' own calls are not recorded. — [raven/agent/tools/registry.py:992](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/agent/tools/registry.py#L992); [raven/acp_client/permissions.py:5-6](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/acp_client/permissions.py#L5-L6) (verified)
  - *To reach the next level:* Third-party sub-agent actions and config changes are not recorded as audit events.
- **D L1:** On by default under ~/.raven/traces, writable by the agent's tools; tracing.enabled is a non-sensitive setting the LLM reviewer may approve. — [raven/tracing/config.py:66](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/tracing/config.py#L66); [raven/config/self_surface.py:662](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/self_surface.py#L662) (verified)
  - *To reach the next level:* Record should be written by a component the model cannot control or disable.
- **B L1:** Write failures are swallowed and the action proceeds. — [raven/tracing/store.py:280](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/tracing/store.py#L280) (verified)
  - *To reach the next level:* Errors should be surfaced and records durably flushed per action.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

Each turn is capped at 40 tool iterations, shell commands time out (60 s default, 600 s max) and their whole process group is killed, model calls have timeouts, and sub-agents are limited to 8 at once and 30 spawns an hour. There is no token or spend cap, sub-agents get their own iteration budgets, and the iteration cap (up to 200) is a setting the LLM reviewer may raise. Long playbook runs and scheduled tasks can continue for days.

- **S L2:** Iteration cap plus per-exec timeout with process-group kill; no cost/token cap. — [raven/config/schema.py:231](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/schema.py#L231); [raven/sandbox/direct_executor.py:209](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/sandbox/direct_executor.py#L209); searched `rg -n -i 'max_cost|cost_limit|spend_limit|max_spend|budget_usd|usd_budget'` in `raven` → 0 hits (No spend or cost cap exists.) (verified)
  - *To reach the next level:* No token/cost cap or rate limits on side-effecting tools.
- **C L2:** Top-level loop and tool timeouts; sub-agents have concurrency and spawn-rate caps but separate iteration budgets. — [raven/config/schema.py:235](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/schema.py#L235); [raven/config/schema.py:242](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/schema.py#L242) (verified)
  - *To reach the next level:* Sub-agents do not draw on the parent's budget.
- **D L1:** maxToolIterations (to 200) is a non-sensitive self-config setting the smart reviewer may approve. — [raven/config/self_surface.py:186](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/config/self_surface.py#L186) (verified)
  - *To reach the next level:* The model should not be able to raise its own limits.
- **B L1:** No spend ceiling; /stop cancels a session's sub-agents but long runs and scheduled work continue. — [raven/cli/gateway_commands.py:941](https://github.com/evermind-ai/raven/blob/3632e6040c7038a60ec418ce39ccae185c72c19f/raven/cli/gateway_commands.py#L941) (verified)
  - *To reach the next level:* Tight per-run time and cost ceilings are absent.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web_fetch/browser auto-allowed (raven/permissions/rules.py:97, :109) · [B] sensitive data/systems: read_file auto-allowed with no fence (raven/permissions/rules.py:89, raven/config/schema.py:1316) · [C] state change / egress: web_fetch egress auto-allowed; pushes/installs approved by the LLM reviewer (raven/permissions/judge.py:32-33) · Same default session? Yes

## Highest-impact improvements
1. Harden plugin loading so third-party code needs an explicit, informed install. — C7 D L0→L3, +0.150 before caps (Playbook 3)
2. Route web_fetch, browser navigation and reads outside the workspace through approval once untrusted content is in the session. — C5 S L1→L2, +0.075 before caps (Playbook 1)
3. Ship permissions.mode=ask, or restrict the smart reviewer to reversible workspace-local actions. — C2 S L1→L3, +0.150 before caps (Playbook 5)
4. Default the sandbox backend to boxlite with network off or allowlisted. — C4 D L0→L2, +0.100 before caps (Playbook 3)
5. Add a per-run token/spend cap shared with sub-agents. — C10 S L2→L3, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The MCP stdio server environment (env=None) is inferred to be the MCP Python SDK's default safe-variable set; the SDK itself was not read.
- Product agents under agents/ (Raven-Code, Raven-Oncall, etc.), the experimental Curator, the A2A face (off by default), channel adapters and the sentinel/cron engines were sampled, not fully reviewed; Raven-Code additionally auto-loads AGENTS.md/CLAUDE.md/CONTEXT.md from the project.
- G2 was not applied to C2/C4 for one plugin-loading path; that path is scored once, under C6 and C7.
- No text aimed at AI reviewers was found in the repository.
