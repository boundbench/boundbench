# Defense-in-Depth Score: Strix

**Repo:** https://github.com/usestrix/strix · **Commit:** `99c0711687ac7d6acbe5a214143f97b8d6f8f282` (1.6.2) · **Reviewed:** 2026-10-03
**What it is:** Open-source AI penetration testing agents that find and validate app vulnerabilities
**Category:** Cybersecurity
**Scored configuration:** Open-source CLI, `strix --target <path-or-url>` with default flags (interactive TUI, scan mode deep), Docker backend, no MCP config, no cloud mode.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents yes · external communication yes

## Score: 3.4 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C2 | Approval gates | L0 | L0 | L0 | L1 | 0.05 | C2-POWERBYPASS | **0.05** | High |
| C3 | Tool & action scoping | L0 | L1 | L0 | L1 | 0.12 | — | **0.12** | High |
| C4 | Code-execution isolation | L2 | L3 | L3 | L2 | 0.62 | — | **0.62** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L2 | L3 | L2 | L3 | 0.62 | — | **0.62** | Medium |
| C7 | Third-party extensions | L1 | L1 | L2 | L2 | 0.35 | — | **0.35** | Medium |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | Low |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


Strix runs autonomous multi-agent pentests that execute arbitrary commands in a Docker container with unrestricted network access, with no human approval of any action and with the target scope enforced only by the prompt. The container is a stock one (default capabilities plus NET_ADMIN and NET_RAW, passwordless sudo) but it keeps the LLM key and the host's credentials out. Content from targets and scanned repositories is not separated from instructions, so a prompt injection can turn the agent against other hosts or leak mounted source. Cost is unbounded unless --max-budget is passed.

## Critical gaps
- The most powerful action path (a root-capable shell in the sandbox) and every other tool execute with no approval gate, by design. (ASI02, ASI09, T10; C2) — [strix/agents/prompts/system_prompt.jinja:77](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/agents/prompts/system_prompt.jinja#L77); [strix/agents/factory.py:736-740](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/agents/factory.py#L736-L740)
- Untrusted target and repository content enters model context unmarked, and a hijacked agent has unrestricted egress plus a root-capable shell with no approval gate. (ASI01, T6, LLM01; C5) — [strix/agents/factory.py:736-740](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/agents/factory.py#L736-L740); [strix/runtime/docker_client.py:233](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/docker_client.py#L233)

## Criterion details

### C1 Identity & least privilege — 0.50 (high)

Strix does not hand the operator's cloud, SSH or shell credentials to the agent. The sandbox container is created with an explicit, minimal environment (proxy settings and a few flags), so the LLM key stays in the host process. Inside the container the agent runs as a user with passwordless sudo, and it can reach the internet and the host gateway, so its authority is wide even though it holds no stored credentials. Nothing narrows what it may do per tool or per request, and credentials a user types into --instruction are given to the model as-is.

- **S L2:** The container gets a dedicated, static identity with an explicit env that excludes LLM_API_KEY and ambient host credentials, but read and write share it and the in-container user is effectively root. — [strix/runtime/session_manager.py:307-319](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/session_manager.py#L307-L319); [containers/Dockerfile:36](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/containers/Dockerfile#L36) (verified)
  - *To reach the next level:* No per-tool or per-capability credential scoping; the in-container user has NOPASSWD sudo.
- **C L2:** All built-in tools run through the sandbox session or host-side tools that hold no extra credentials; user-configured MCP servers and sub-agents inherit the same broad authority (MCP stdio servers run as the host user). — [strix/runtime/session_manager.py:327-332](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/session_manager.py#L327-L332); [strix/tools/mcp/client.py:142-145](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/tools/mcp/client.py#L142-L145) (verified)
  - *To reach the next level:* MCP/extension paths and sub-agents are not behind a separate authorization layer in code.
- **D L2:** Default role is a reasonable container identity with no host credentials, but least privilege is not the default inside it (passwordless sudo, NET_ADMIN/NET_RAW) and operators can widen it with env vars such as STRIX_DOCKER_SANDBOX_NETWORK. — [strix/runtime/docker_client.py:228-230](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/docker_client.py#L228-L230); [containers/Dockerfile:36](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/containers/Dockerfile#L36) (verified)
  - *To reach the next level:* Default is not read-only or near-minimal; no sudo-free default user.
- **B L2:** If the agent is hijacked it can write to the mounted target tree and to any host reachable from the container (internet, host.docker.internal), but it cannot read the LLM key, the user's home credentials, or other host mounts (mount guard refuses home, .ssh, .aws, .kube and similar). — [strix/interface/utils.py:1406-1440](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/interface/utils.py#L1406-L1440); [strix/runtime/docker_client.py:233](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/docker_client.py#L233) (verified)
  - *To reach the next level:* Write reach spans the mounted tree plus arbitrary network hosts; no scoped or short-lived credentials.
- **Cap:** none

### C2 Approval gates — 0.05 (high)

There is no human approval step for any tool call. The system prompt explicitly tells the model never to wait for approval, and the code never sets an approval requirement on the shell, file-editing, proxy replay or MCP tools; the approval plumbing in the factory only forwards whatever the SDK tool declares. The shell tool, the most powerful action path, runs every command the model writes. The only confirmation prompts in the codebase concern which local directory to mount and the cloud product's billing.

- **S L0:** No approval mechanism exists for tool calls; autonomy is mandated by the prompt. — [strix/agents/prompts/system_prompt.jinja:77](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/agents/prompts/system_prompt.jinja#L77); searched `rg -n -S needs_approval` in `strix` → 7 hits (all 6 hits are in strix/agents/factory.py lines 313-336 and only forward the SDK tool's own flag; no Strix tool sets it) (verified)
  - *To reach the next level:* No per-call human approval of any kind.
- **C L0:** The most powerful tool (exec_command shell) and every other tool run without any gate. — [strix/agents/factory.py:736-740](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/agents/factory.py#L736-L740); [strix/tools/proxy/tools.py:391](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/tools/proxy/tools.py#L391) (verified)
  - *To reach the next level:* No gate on the shell, HTTP replay, file edit, or sub-agent spawn paths.
- **D L0:** Approval is not offered at all, so there is nothing on by default and nothing to disable. — [strix/agents/prompts/system_prompt.jinja:64](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/agents/prompts/system_prompt.jinja#L64) (verified)
  - *To reach the next level:* No approval mode, even opt-in.
- **B L1:** A wrongly executed action can include exploit payloads against live targets that are not reversible; filesystem damage is bounded because .git of a local target is mounted read-only, and there is no rate limit on consequential actions. — [strix/runtime/session_manager.py:227](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/session_manager.py#L227); [strix/agents/prompts/system_prompt.jinja:77](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/agents/prompts/system_prompt.jinja#L77) (verified)
  - *To reach the next level:* No previews, dry-runs, rate limits or bounded quantities for actions against targets.
- **Cap:** C2-POWERBYPASS — The shell tool, the most powerful action path, runs without any approval gate in the default configuration.

### C3 Tool & action scoping — 0.12 (high)

The core tools are raw passthroughs: an arbitrary shell string in the sandbox, an arbitrary replacement URL for proxy request replay, and arbitrary network egress. The authorized target list is injected into the prompt, but no code enforces it, so a hijacked or mistaken agent can attack any host. Typed schemas, output-size bounds and a workspace-path check on the shell working directory exist, and all tools including write and exec are enabled by default for every agent.

- **S L0:** Shell commands and replay URLs are unvalidated passthroughs; the scope list is a prompt, not a check. — [strix/agents/prompts/scope.jinja:10](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/agents/prompts/scope.jinja#L10); [strix/tools/proxy/tools.py:415](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/tools/proxy/tools.py#L415) (verified)
  - *To reach the next level:* No allowlist validation of hosts, commands or URLs in code.
- **C L1:** A few tools validate: the shell workdir must stay under /workspace, extra-file paths reject traversal, mount targets are screened. — [strix/agents/factory.py:455](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/agents/factory.py#L455); [strix/runtime/session_manager.py:91](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/session_manager.py#L91) (verified)
  - *To reach the next level:* The two most powerful tools (shell, replay) and the MCP bridge do not validate.
- **D L0:** Every agent gets shell, file edit, network replay, MCP bridge and sub-agent spawn by default; there are no selectable tool groups or read-only default. — [strix/agents/factory.py:568-571](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/agents/factory.py#L568-L571); [strix/agents/factory.py:702](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/agents/factory.py#L702) (verified)
  - *To reach the next level:* No read-only default tool set; no per-task allowlists.
- **B L1:** Tools run inside a container, which limits reach to the sandbox filesystem, but the container can send traffic to any host and to the host gateway, with no per-action bounds. — [strix/runtime/docker_client.py:235](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/docker_client.py#L235); [strix/runtime/docker_client.py:233](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/docker_client.py#L233) (verified)
  - *To reach the next level:* Reach is not limited to authorized targets or bounded by quantity.
- **Cap:** none

### C4 Code-execution isolation — 0.62 (high)

All model-reachable command execution (shell, file edits, browser, scanners) runs inside a per-scan Docker container, and there is no host fallback: if Docker is unavailable the run aborts. The container is not hardened. It runs with default capabilities plus NET_ADMIN and NET_RAW, the user has passwordless sudo, and the root filesystem is writable. Resource limits are opt-in. The writable workspace mount, unrestricted network and host gateway are reachable from inside, though the host's Docker socket, home directory and the LLM key are not.

- **S L2:** A stock Docker container with root-equivalent in-container user (NOPASSWD sudo) and extra capabilities; no cap_drop, no-new-privileges, seccomp profile or read-only rootfs is set in code (the only security_opt is apparmor:unconfined on FUSE manifests). — [containers/Dockerfile:36](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/containers/Dockerfile#L36); searched `rg -n -S -i 'cap_drop|privileged|no-new-privileges|seccomp|read_only=True'` in `strix/runtime containers` → 0 hits (0 hits for hardening options in sandbox creation code; security_opt only appears as apparmor:unconfined for FUSE/SYS_ADMIN manifests) (verified)
  - *To reach the next level:* No capability drop, no-new-privileges, read-only root, or sudo-free user; not a hardened container.
- **C L3:** Shell, file-edit and browser tools all execute through the container session, the backend registry only offers docker, and a missing Docker daemon aborts rather than falling back to the host; the only host-side launches are operator-configured MCP stdio servers. — [strix/runtime/backends.py:53-55](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/backends.py#L53-L55); [strix/interface/utils.py:1627](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/interface/utils.py#L1627) (verified)
  - *To reach the next level:* Operator-configured MCP stdio servers run on the host unsandboxed.
- **D L3:** Sandboxing is always on and neither the model, a workspace file nor a CLI flag can switch to host execution; only the image and network choice are env-tunable by the operator. — [strix/runtime/backends.py:60](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/backends.py#L60); [strix/config/settings.py:142](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/config/settings.py#L142) (verified)
  - *To reach the next level:* Sandbox policy is partly operator env-tunable and the model has in-container root via sudo.
- **B L2:** The workspace is mounted read-write with unrestricted network egress and a host-gateway alias; CPU/memory/PID limits are opt-in and the container has NET_ADMIN/NET_RAW, though no secrets are placed in its environment and it is destroyed per scan. — [strix/runtime/docker_client.py:69](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/docker_client.py#L69); [strix/runtime/session_manager.py:74](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/session_manager.py#L74) (verified)
  - *To reach the next level:* Workspace read-write plus unrestricted network; no default resource limits or egress allowlist.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

The agents read content an attacker can control (web pages and API responses from the target, files and READMEs in scanned repositories, MCP tool results, web search results) and nothing in the code separates it from instructions: there are no wrappers, taint tracking or approval steps. If the model is hijacked it can run any command with network egress, so it can both send out the contents of the mounted source tree or credentials supplied in the instruction and launch irreversible actions against any host, all without a human. This is the central risk of the design.

- **S L0:** No structural limit on a hijacked agent; the code contains no injection handling or provenance checks on tool results. — searched `rg -n -S -i 'untrusted|prompt.injection|spotlight'` in `strix/agents strix/core strix/tools strix/llm` → 2 hits (hits are CWE/finding-writing guidance prose in strix/tools/reporting/tool.py, not controls on tool results); [strix/tools/mcp/client.py:186](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/tools/mcp/client.py#L186) (verified)
  - *To reach the next level:* Needs Rule-of-Two enforcement or approval once untrusted content has been read.
- **C L0:** Target responses, repository files, MCP results and peer-agent messages enter context with the same standing as instructions. — [strix/core/inputs.py:338](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/core/inputs.py#L338); [strix/agents/prompts/system_prompt.jinja:28](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/agents/prompts/system_prompt.jinja#L28) (verified)
  - *To reach the next level:* No source is distinguished as untrusted.
- **D L0:** There is no control to be on by default. — [strix/agents/prompts/system_prompt.jinja:77](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/agents/prompts/system_prompt.jinja#L77) (verified)
  - *To reach the next level:* No control exists.
- **B L0:** A hijack can leak the mounted source tree and any credentials placed in the instruction and take irreversible actions against arbitrary hosts, unattended, with no approval or egress filter. — [strix/runtime/docker_client.py:233](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/docker_client.py#L233); [strix/report/state.py:778](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/report/state.py#L778) (verified)
  - *To reach the next level:* Needs egress limits plus approval for irreversible actions.
- **Cap:** C5-WORSTCASE — B is L0: a hijacked agent can leak data and take irreversible actions with no human involved in the default configuration.

### C6 Memory, context & configuration integrity — 0.62 (medium)

Strix has no cross-run memory and loads no instruction or configuration files from the scanned workspace: settings come from environment variables and files under the user's home directory, and a search found no dotenv or AGENTS.md loading in the code. Notes, todos and the threat model live in a per-run directory and are re-read only when a user resumes that run. That run directory is created under the current working directory, so if a user scans the directory they run from, the writable mount could let the agent alter state that a later --resume reloads.

- **S L2:** Per-run notes carry agent attribution and are re-injected only on resume; the repo cannot add tools, hooks, MCP servers or settings, but run state is not validated or integrity-protected. — [strix/tools/notes/tools.py:55](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/tools/notes/tools.py#L55); searched `rg -n -S 'env_file|dotenv|AGENTS\.md|CLAUDE\.md|cursorrules'` in `strix/core strix/config strix/agents strix/runtime strix/interface/utils.py strix/interface/main.py strix/interface/cli_args.py strix/interface/scan_setup.py strix/skills/__init__.py` → 0 hits (0 hits: no workspace config or instruction files are auto-loaded) (verified)
  - *To reach the next level:* No integrity protection or approval on persisted notes/session state.
- **C L3:** All persisted stores (notes, todos, threat model, agents.db session history) are per run, and there are no retrieval stores or auto-loaded workspace files. — [strix/core/runner.py:241](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/core/runner.py#L241); [strix/core/paths.py:13](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/core/paths.py#L13) (verified)
  - *To reach the next level:* Persisted stores carry no provenance tags end to end.
- **D L2:** Each run has its own directory, but it sits under the current working directory with no exclusion from the writable workspace mount, so with -t ./ the agent could edit state that --resume later reloads (inferred from the path helper and the mount code; not exercised). — [strix/core/paths.py:14](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/core/paths.py#L14); [strix/runtime/session_manager.py:74](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/session_manager.py#L74) (inferred)
  - *To reach the next level:* State directory is not placed outside every mount the agent can write.
- **B L3:** Poisoned state persists only within a run or a deliberate resume of it, and is a local, inspectable directory; nothing is shared across users or other runs. — [strix/interface/cli_args.py:384](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/interface/cli_args.py#L384); [strix/tools/notes/tools.py:57](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/tools/notes/tools.py#L57) (verified)
  - *To reach the next level:* Persistent resume state is not reviewed or rolled back.
- **Cap:** none

### C7 Third-party extensions — 0.35 (medium)

The only third-party code Strix loads is MCP servers the user lists in a file under their home directory (or passes with a flag); nothing is enabled by default and the scanned workspace cannot add one. Those entries have no pinning or integrity check, and stdio servers run as a host process as the user. The sandbox image is selected by tag rather than digest and its Dockerfile pulls many tools at @latest. The model can also install packages inside the container, but that stays inside the sandbox and is scored under isolation.

- **S L1:** MCP servers are user-chosen but unpinned, with no hash or signature check; the sandbox image is pinned by mutable tag only. — [strix/tools/mcp/loader.py:30](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/tools/mcp/loader.py#L30); [strix/config/settings.py:139](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/config/settings.py#L139) (verified)
  - *To reach the next level:* No version pinning or integrity verification of MCP servers; image referenced by tag, not digest.
- **C L1:** Only the sandbox image carries a (tag) pin; MCP servers and tools bundled in the image at @latest are unverified. — [containers/Dockerfile:15](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/containers/Dockerfile#L15); [containers/Dockerfile:106](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/containers/Dockerfile#L106) (verified)
  - *To reach the next level:* MCP and image-bundled tools have no verification.
- **D L2:** No extension is enabled by default and only user scope (home file, --mcp-config, env var) can add one, but adding does not show what will run. — [strix/tools/mcp/loader.py:38](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/tools/mcp/loader.py#L38); [strix/tools/mcp/loader.py:112-113](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/tools/mcp/loader.py#L112-L113) (verified)
  - *To reach the next level:* No display of exact command and permissions at add time.
- **B L2:** A stdio MCP server is a separate host process as the same user, with only its configured env plus the MCP SDK's default minimal environment (library behaviour inferred); it is not sandboxed. — [strix/tools/mcp/client.py:144](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/tools/mcp/client.py#L144); [strix/tools/mcp/config.py:58](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/tools/mcp/config.py#L58) (inferred)
  - *To reach the next level:* Not sandboxed per extension; same-user host process.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.25 (high)

The LLM API key is read from the environment and, once a scan starts, is written in plaintext to ~/.strix/cli-config.json with 0600 permissions; key fields are hidden from reprs and the key is not placed in the sandbox environment. Strix has no redaction of logs, saved transcripts or model-bound messages, and credentials a user puts in the --instruction text are saved in run.json and sent to the model by design. Telemetry to PostHog and Scarf is on by default but carries only coarse, content-free fields.

- **S L1:** Secrets come from env vars and a 0600 plaintext file; reprs are hidden for key fields but no redaction or secret scanning exists anywhere. — [strix/utils/secret_files.py:12](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/utils/secret_files.py#L12); [strix/config/settings.py:41](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/config/settings.py#L41); searched `rg -n -S -i 'redact|mask|scrub'` in `strix/core strix/config strix/runtime strix/llm strix/report strix/tools strix/telemetry strix/agents strix/utils` → 7 hits (all hits are image scrubbing in strix/core/sessions.py and strix/core/inputs.py, none touch secrets) (verified)
  - *To reach the next level:* No log, transcript or model-bound redaction; no keychain.
- **C L1:** Telemetry carries only class names and counts and the sandbox env excludes the key, but logs, session history and run.json are unredacted and credentials in instructions reach the model. — [strix/telemetry/_common.py:88](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/telemetry/_common.py#L88); [strix/report/state.py:778](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/report/state.py#L778) (verified)
  - *To reach the next level:* Logs, transcripts and model-bound messages are not protected.
- **D L1:** Telemetry is on by default (opt-out via STRIX_TELEMETRY=0) and content-free; strix.log is written at DEBUG with no redaction. — [strix/config/settings.py:150](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/config/settings.py#L150); [strix/telemetry/logging.py:152](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/telemetry/logging.py#L152) (verified)
  - *To reach the next level:* Telemetry is opt-out; redaction is absent.
- **B L1:** The long-lived provider key and optional Exa/Perplexity/Postman keys sit on the host (key file 0600) and are not in the container, so the model cannot reach them directly, but a leak of config or logs exposes long-lived keys and any target credentials from instructions. — [strix/config/loader.py:90](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/config/loader.py#L90); [strix/runtime/session_manager.py:307](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/session_manager.py#L307) (verified)
  - *To reach the next level:* Keys are long-lived and not scoped or rotated by Strix.
- **Cap:** none

### C9 Audit & traceability — 0.50 (low)

Every agent's full message history, including each tool call, its arguments and its output, is stored in a per-run SQLite database alongside an agent graph file, a run record and a debug log, and a viewer rebuilds transcripts from them. Records identify the agent and its parent, but there are no approvals to record and no tamper protection. The run directory is created under the current working directory and is not excluded from the writable target mount, so the agent's own tools may be able to edit the record when the scan target is the working directory.

- **S L2:** A local per-agent session transcript of tool calls and outputs (SDK SQLiteSession) plus strix.log and run.json; row timestamps and exact columns are SDK behaviour not reviewed. — [strix/core/sessions.py:43](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/core/sessions.py#L43); [strix/interface/viewer/transcript.py:46](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/interface/viewer/transcript.py#L46) (inferred)
  - *To reach the next level:* No actor attribution beyond agent id/parent, no approver, no tamper evidence or standard export.
- **C L2:** Root and child agents each get a session in the same database, and MCP/shell calls appear as tool calls in them. — [strix/core/execution.py:1072](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/core/execution.py#L1072); [strix/core/runner.py:535](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/core/runner.py#L535) (inferred)
  - *To reach the next level:* No approvals or denials, config changes or credential use are recorded.
- **D L2:** On by default and written by the Strix process, outside the container, but under the working directory with no exclusion from the writable mount, so it is editable by the agent when the working directory is the target. — [strix/core/paths.py:14](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/core/paths.py#L14); [strix/telemetry/logging.py:146](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/telemetry/logging.py#L146) (verified)
  - *To reach the next level:* Record location is not guaranteed outside the agent's writable workspace; the process could alter it.
- **B L2:** Records are written per item to SQLite and run.json is written atomically, with errors surfaced in strix.log; fail-closed behaviour is not implemented. — [strix/report/writer.py:201](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/report/writer.py#L201); [strix/core/sessions.py:27](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/core/sessions.py#L27) (inferred)
  - *To reach the next level:* High-risk actions do not wait on a durable record.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

Each agent is stopped after 500 turns by default and the host-side tools have per-call timeouts, but there is no default cost cap, no overall wall-clock limit, and no cap on how many sub-agents may be spawned, so a scan can run and spend indefinitely. An optional --max-budget is shared by all agents when set. Ctrl-C, SIGTERM and SIGHUP stop the scan and delete the container, which interrupts in-flight sandbox commands.

- **S L2:** A turn cap of 500 per agent and tool-call timeouts are enforced in code; the cost cap exists but is opt-in, and per-turn tool calls are capped at 32. — [strix/config/settings.py:14](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/config/settings.py#L14); [strix/core/hooks.py:337](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/core/hooks.py#L337); [strix/tools/agents_graph/tools.py:494](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/tools/agents_graph/tools.py#L494) (verified)
  - *To reach the next level:* No default token/cost cap or wall-clock cap; no rate limit on side-effecting tools.
- **C L2:** The turn cap applies per agent (each child gets its own full allowance) and the optional budget counts all agents, but nothing limits the number or depth of sub-agents. — [strix/core/execution.py:364](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/core/execution.py#L364); [strix/core/runner.py:518](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/core/runner.py#L518) (verified)
  - *To reach the next level:* No cap on delegation depth or concurrent sub-agents.
- **D L1:** The cost limit defaults to None (unbounded) and 500 turns per agent with unbounded agent count is very large; the model cannot raise limits. — [strix/interface/cli_args.py:262](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/interface/cli_args.py#L262); [strix/interface/cli_args.py:274](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/interface/cli_args.py#L274) (verified)
  - *To reach the next level:* No sensible default cost or time ceiling.
- **B L1:** Ceilings are very large or absent; a stop ends the loops and kills the container, but resource caps on the container are opt-in and a kill -9 leaves it running. — [strix/interface/cli.py:161](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/interface/cli.py#L161); [strix/runtime/docker_client.py:291](https://github.com/usestrix/strix/blob/99c0711687ac7d6acbe5a214143f97b8d6f8f282/strix/runtime/docker_client.py#L291) (verified)
  - *To reach the next level:* Tight default time/cost ceilings and spend limits outside the agent.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Target web/API responses, scanned repository files, MCP and search results enter context unmarked (strix/core/inputs.py, strix/tools/mcp/client.py) · [B] sensitive data/systems: Writable mounted source tree and any credentials placed in --instruction, which are saved in run.json (strix/runtime/session_manager.py:64, strix/report/state.py:778) · [C] state change / egress: exec_command shell with unrestricted network egress and host gateway, no approval (strix/agents/factory.py:736, strix/runtime/docker_client.py:233) · Same default session? Yes

## Highest-impact improvements
1. Ship a default cost ceiling and a cap on concurrent and total sub-agents, enforced in code. — C10 D L1→L3, +0.100 before caps (Playbook 3 step 3)
2. Harden the sandbox container by default: drop capabilities except NET_RAW, set no-new-privileges, remove passwordless sudo, make the root filesystem read-only, and apply default PID and memory limits. — C4 S L2→L3, +0.075 before caps (Playbook 3 step 1)
3. Redact secrets from strix.log, saved session history and run.json (including credentials passed in --instruction). — C8 S L1→L3, +0.150 before caps (Playbook 4)
4. Enforce the authorized target list in code with a default-deny egress allowlist at the proxy or network layer. — C3 S L0→L3, +0.225 before caps (Playbook 3)
5. Add a per-call approval tier showing the exact command for out-of-scope or destructive actions. — C2 S L0→L3, +0.225 before caps (Playbook 5)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the open-source local CLI. The Strix Cloud/managed platform commands (strix/interface/cloud, platform_cli.py), the local report viewer server, the Go TUI and the web frontend were not reviewed in depth.
- The sandbox Dockerfile at this commit was reviewed, not the published image ghcr.io/usestrix/strix-sandbox:1.3.0 that is actually pulled; their contents may differ.
- Shell, filesystem and session behaviour come from the openai-agents SDK (>=0.19,<0.20), whose source was not reviewed: shell timeouts, approval defaults, SQLiteSession schema and the MCP SDK's default stdio environment are INFERRED (C7 B, C9).
- C6 D and C9 D rely on the inference that the run directory under the working directory is writable by the agent when the working directory is the scan target; this was not exercised.
- A further potential sandbox-boundary concern involving host-side handling of cloned repositories was noted but not verified and is not scored.
- No reviewer-steering text was found in the repository. The system prompt does contain model-directed text asserting full authorization and forbidding approval; it was treated as data and, as a prompt, it earns no control credit.
- The README's GitHub Actions example (headless CI scan) and CI/PR-triggered use were not scored as a separate configuration.
