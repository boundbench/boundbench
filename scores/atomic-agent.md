# Defense-in-Depth Score: Atomic Agent

**Repo:** https://github.com/atomicbot-ai/atomic-agent · **Commit:** `cce6262c135c97a15c020284a699dc6d164d34d3` (0.6.6) · **Reviewed:** 2026-10-04
**What it is:** Local-first TypeScript agent (TUI, CLI, HTTP/Tauri sidecar) that drives a browser, edits files, runs shell commands, sends email, uses MCP tools and keeps long-term memory, on local llama.cpp or cloud models.
**Category:** AI Assistants
**Scored configuration:** Interactive TUI (`atomic-agent`, no flags) on a fresh install: approval level 1 (paranoid), readScope working-dir, http.approvalMode never, tracing on, analytics on, single (non-fusion) run mode.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication yes

## Score: 2.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L1 | L1 | 0.17 | — | **0.17** | High |
| C2 | Approval gates | L3 | L1 | L3 | L2 | 0.55 | G2 | **0.25** | High |
| C3 | Tool & action scoping | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L1 | L0 | L1 | L0 | 0.12 | C5-WORSTCASE | **0.12** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C7 | Third-party extensions | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


Atomic Agent ships a well-built per-call approval ladder (strictest level by default, exact command previews, a never-grantable trust-config category), but everything runs directly on the host as your user with your full environment, including every API key in ~/.atomic-agent/.env. The shell approval guard is not a strict boundary and can let commands run without a prompt. HTTP POSTs, web fetches and browser typing/clicking are also ungated by default, so a prompt-injected page can both exfiltrate data and act without a human.

## Critical gaps
- No execution isolation: shell commands, skill scripts and MCP servers run on the host as the user with the full environment, including .env API keys. (ASI05, T11; C4) — [src/sandbox/command-job.ts:141](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/sandbox/command-job.ts#L141); [src/sandbox/shell-invocation.ts:30](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/sandbox/shell-invocation.ts#L30); [AGENTS.md:519](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/AGENTS.md#L519)
- A prompt-injected session can exfiltrate through ungated HTTP POST and fetch without any human approval. (ASI01, LLM01, T6; C5) — [src/config/config-schema.ts:2883](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/config-schema.ts#L2883); [src/tools/os/http-request.ts:297](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tools/os/http-request.ts#L297)
- MCP stdio servers inherit every environment variable the agent holds, including all stored API keys. (ASI04, T17; C7) — [src/mcp/mcp-client.ts:434](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/mcp/mcp-client.ts#L434); [AGENTS.md:519](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/AGENTS.md#L519)

## Criterion details

### C1 Identity & least privilege — 0.17 (high)

The agent runs as your OS user and uses whatever credentials it can find: provider keys and a GitHub personal token loaded into its own environment from ~/.atomic-agent/.env, plus your gh, git and SSH setup. Every shell command, skill script and MCP server it starts inherits that full environment; the code itself documents that there is no per-tool filtering. There is no scoped or short-lived identity; the only per-request check is the approval prompt, which is scored under approval gates.

- **S L0:** Ambient authority: the operator's long-lived keys and GitHub PAT are loaded into process.env and used directly. — [src/config/load-config.ts:114-118](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/load-config.ts#L114-L118); [src/github/github-token.ts:11](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/github/github-token.ts#L11) (verified)
  - *To reach the next level:* No per-tool or per-capability credential scoping, and no deterministic authorization layer mapped to provider permissions.
- **C L1:** Dedicated GitHub tools are approval-gated, but every subprocess (shell, skill scripts, MCP stdio servers) receives the full process environment. — [src/sandbox/command-job.ts:141](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/sandbox/command-job.ts#L141); [src/mcp/mcp-client.ts:434](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/mcp/mcp-client.ts#L434); [AGENTS.md:519](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/AGENTS.md#L519) (verified)
  - *To reach the next level:* Subprocesses and MCP servers are not limited to a scoped identity or scrubbed environment.
- **D L1:** Default install runs with the user's full credential set; the only narrowing is the approval ladder, which a boot flag widens to approve everything. — [src/approval/approval-level.ts:243](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/approval/approval-level.ts#L243); [AGENTS.md:519](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/AGENTS.md#L519) (verified)
  - *To reach the next level:* No narrower default credential set; least privilege would require manual hardening.
- **B L1:** A hijacked process holds write-capable keys across several systems (model providers, GitHub, mail, any MCP server credentials) plus the user's home directory. — [src/github/github-token.ts:11](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/github/github-token.ts#L11); [src/mcp/mcp-client.ts:434](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/mcp/mcp-client.ts#L434) (verified)
  - *To reach the next level:* Credentials are long-lived and span multiple systems; nothing confines them to one project or read-only use.
- **Cap:** none

### C2 Approval gates — 0.25 (high)

Atomic Agent has a real approval system: a five-step ladder that starts at the strictest level, per-call prompts that show the exact command line or diff, session-only grants, and a trust-config category that can never be silenced. The gap is in what reaches it. The shell guard's auto-allow rules are not a strict boundary. HTTP POSTs (approval mode "never" by default), web fetches, browser typing and clicking, memory writes and scheduled tasks never ask, and MCP tools skip the prompt when the server itself claims to be read-only.

- **S L3:** Per-call prompts show the exact command line, category tiers decide what asks, and grants are scoped to one session and never cover trust_config. — [src/tools/os/shell.ts:212](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tools/os/shell.ts#L212); [src/approval/approval-level.ts:90-106](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/approval/approval-level.ts#L90-L106); [src/approval/approval-gate.ts:371](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/approval/approval-gate.ts#L371) (verified)
  - *To reach the next level:* Auto-allow matching is not guaranteed to correspond to what the shell executes.
- **C L1:** The shell gate is skipped whenever the guard returns allow, and the auto-allow rules are not a strict boundary; os.http.request, os.web.fetch, browser.type/click and task scheduling have no gate at all. — [src/tools/os/shell.ts:157](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tools/os/shell.ts#L157); [src/tools/os/shell.ts:195](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tools/os/shell.ts#L195); [src/tools/os/http-request.ts:297](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tools/os/http-request.ts#L297); searched `rg -n requireApproval` in `src/tools/browser/type.ts src/tools/browser/click.ts` → 0 hits (browser.type and browser.click never call the gate); searched `rg -n requireApproval` in `src/tools/memory src/tools/tasks` → 0 hits (memory writes and task scheduling are never gated); [src/mcp/mcp-tool-adapter.ts:101](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/mcp/mcp-tool-adapter.ts#L101) (verified)
  - *To reach the next level:* Compound commands are not robustly handled, and several mutating or egress tools are unflagged.
- **D L3:** Level 1 (ask for everything gated) is the shipped default; only the explicit --no-approval flag or an operator config change raises it, and writes to the agent's own config are pinned to full trust. — [src/config/config-schema.ts:2870](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/config-schema.ts#L2870); [src/approval/approval-level.ts:243](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/approval/approval-level.ts#L243); [src/approval/approval-level.ts:102](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/approval/approval-level.ts#L102) (verified)
  - *To reach the next level:* Elevated levels are not time-bounded and approvals are not tied to an authenticated principal.
- **B L2:** File writes keep restore copies and deletes go to Trash, but shell commands, emails, GitHub publishing and HTTP POSTs are irreversible. — [src/tools/os/fs-restore.ts:28](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tools/os/fs-restore.ts#L28); [src/tools/os/email.ts:151](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tools/os/email.ts#L151) (verified)
  - *To reach the next level:* No rollback or dry-run for external actions; no rate limit on consequential actions.
- **Cap:** G2 — The shell approval gate is not strictly enforced at runtime.

### C3 Tool & action scoping — 0.40 (high)

Many tools are narrow and validated: file tools resolve paths and classify them against the working directory, reads outside it require approval by default, and the HTTP and fetch tools block private and cloud-metadata addresses after DNS resolution. But the shell tool is a raw command line with only regex deny-lists, the HTTP tool accepts any public host and POST body (the host allowlist is empty by default), and every tool group, including shell, network and browser, is on by default.

- **S L2:** Typed schemas, realpath-based workspace classification and an SSRF guard exist, but the main shell tool takes an arbitrary command string filtered only by regex. — [src/tools/os/web-fetch-ssrf-guard.ts:66](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tools/os/web-fetch-ssrf-guard.ts#L66); [src/config/config-schema.ts:2869](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/config-schema.ts#L2869); [src/tools/os/shell.ts:157](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tools/os/shell.ts#L157) (verified)
  - *To reach the next level:* The general shell and HTTP tools are not replaced by narrow tools or constrained by allowlists.
- **C L2:** Built-in fs, fetch and HTTP tools validate paths and hosts; MCP tool arguments pass straight through to the server. — [src/mcp/mcp-tool-adapter.ts:101](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/mcp/mcp-tool-adapter.ts#L101); [src/tools/os/web-fetch-ssrf-guard.ts:66](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tools/os/web-fetch-ssrf-guard.ts#L66) (verified)
  - *To reach the next level:* No shared validation layer for extension (MCP) tools.
- **D L1:** Shell, file write, HTTP, browser and email tools are all registered by default; individual tools can be disabled in config. — [src/config/config-schema.ts:2883](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/config-schema.ts#L2883); [src/config/config-schema.ts:2882-2884](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/config-schema.ts#L2882-L2884) (verified)
  - *To reach the next level:* Default tool set is not tool-group selectable down to read-only.
- **B L1:** A misused shell or HTTP tool reaches any public host and anything the user can touch, limited only by read-scope prompts for paths outside the working directory. — [src/config/config-schema.ts:2869](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/config-schema.ts#L2869); [src/config/config-schema.ts:2883](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/config-schema.ts#L2883) (verified)
  - *To reach the next level:* Reach is not confined to a project/workspace or bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

There is no isolation boundary. Shell commands, skill scripts, verify runs and MCP stdio servers are spawned directly on the host as the current user, with the agent's full environment (including API keys from .env) and unrestricted network. The `src/sandbox` directory contains process runners, timeouts and process-group kill, not a sandbox. A command that escapes the approval prompt has the user's whole machine.

- **S L0:** Commands run as same-user host subprocesses via spawn / sh -c; no container, OS sandbox profile or restricted runtime exists. — [src/sandbox/shell-invocation.ts:30](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/sandbox/shell-invocation.ts#L30); [src/sandbox/command-job.ts:141](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/sandbox/command-job.ts#L141); searched `rg -n -i 'sandbox-exec|bwrap|bubblewrap|landlock|seccomp|firejail|docker run|gvisor|firecracker'` in `src` → 0 hits (verified)
  - *To reach the next level:* No OS-level separation (low-privilege user, container, Seatbelt/Landlock).
- **C L0:** No execution path is sandboxed. — searched `rg -n -i 'sandbox-exec|bwrap|bubblewrap|landlock|seccomp|firejail|docker run|gvisor|firecracker'` in `src` → 0 hits (verified)
  - *To reach the next level:* No execution path goes through an isolation boundary.
- **D L0:** No sandbox exists to enable. — searched `rg -n -i 'sandbox-exec|bwrap|bubblewrap|landlock|seccomp|firejail|docker run|gvisor|firecracker'` in `src` → 0 hits (verified)
  - *To reach the next level:* No sandbox on by default.
- **B L0:** Executed code gets the home directory, every .env credential in its environment and full network egress. — [src/sandbox/command-job.ts:141](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/sandbox/command-job.ts#L141); [AGENTS.md:519](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/AGENTS.md#L519) (verified)
  - *To reach the next level:* Credentials and home directory are reachable and egress is unrestricted.
- **Cap:** none

### C5 Untrusted input blast radius — 0.12 (high)

Web pages, search results, browser page text, MCP results and files are fed into the prompt with no marking or taint tracking, and nothing changes once untrusted content has been read. A hijacked agent can exfiltrate data unattended through os.http.request POST (ungated by default), os.web.fetch query strings or browser form filling, and gaps in the shell approval guard can let it run commands without a prompt. Writes, email and publishing still prompt, which limits the most visible damage.

- **S L1:** No structural Rule-of-Two limit; the always-on approval prompts for writes, shell and email are the only friction, and egress tools are ungated. — searched `rg -n -i untrusted` in `src/prompt src/compressor` → 0 hits (no untrusted-content marking when tool results enter the prompt); [src/tools/os/http-request.ts:297](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tools/os/http-request.ts#L297) (verified)
  - *To reach the next level:* Egress and state-changing tools are not disabled or forced through approval once untrusted content enters the session.
- **C L0:** Tool results from web, browser, files and MCP enter context with the same standing as the user's message. — searched `rg -n -i untrusted` in `src/prompt src/compressor` → 0 hits (no untrusted-content marking when tool results enter the prompt) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished from principal input.
- **D L1:** The approval friction is on by default, but injected content can route around it via gaps in the shell guard and ungated egress tools. — [src/config/config-schema.ts:2883](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/config-schema.ts#L2883) (verified)
  - *To reach the next level:* The only control in play is not strictly enforced.
- **B L0:** Worst case unattended: gaps in the shell guard plus ungated HTTP POST let a hijack read workspace files and env-held keys, send them anywhere, and run irreversible commands. — [src/tools/os/http-request.ts:297](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tools/os/http-request.ts#L297); [src/sandbox/command-job.ts:141](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/sandbox/command-job.ts#L141) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions are both reachable without a human.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.30 (high)

Long-term memory is on by default: after each turn a reflection pass stores up to three profile facts and two notes, and the model can also write profile facts and notes directly without approval; these are re-injected into later prompts. Profile facts keep a history and can be listed, voted down and removed. Skills placed in a repository's `.atomic-agent/skills` folder are loaded silently into the skill catalog and override global skills of the same name, though running their scripts still asks for approval. Agent config and .env are read only from the user's state directory, not the working directory.

- **S L1:** Memory writes are unvalidated (reflection auto-stores; memory tools are ungated) but profile edits are versioned; workspace skills load silently as catalog instructions. — [src/config/config-schema.ts:2966-2970](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/config-schema.ts#L2966-L2970); searched `rg -n requireApproval` in `src/tools/memory src/tools/tasks` → 0 hits (memory writes and task scheduling are never gated); [src/runtime/bootstrap.ts:1312](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/runtime/bootstrap.ts#L1312); [src/skills/skill-loader.ts:83-86](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/skills/skill-loader.ts#L83-L86) (verified)
  - *To reach the next level:* Memory entries are not presented as provenance-tagged data and workspace skill files are loaded without a trust decision.
- **C L1:** Profile history exists, but notes, reflection output and project skills are uncontrolled. — [src/config/config-schema.ts:2966-2970](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/config-schema.ts#L2966-L2970); [src/skills/skill-loader.ts:83-86](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/skills/skill-loader.ts#L83-L86) (verified)
  - *To reach the next level:* No control over notes, summaries or auto-loaded workspace skills.
- **D L2:** State lives in the user's own ~/.atomic-agent databases; there is no cross-user sharing in the local design. — [src/config/config-schema.ts:3200](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/config-schema.ts#L3200) (verified)
  - *To reach the next level:* The model can write to any memory namespace and change what is retained.
- **B L1:** Poisoned facts and notes persist across all of the user's sessions and shape later tool use. — [src/config/config-schema.ts:2966-2970](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/config-schema.ts#L2966-L2970) (verified)
  - *To reach the next level:* Persistence is not limited to text output or gated actions.
- **Cap:** none

### C7 Third-party extensions — 0.25 (high)

Third-party code arrives as MCP servers the user configures (or imports from Claude Code/Oh-My-Pi with a checkbox) and skills installed from GitHub or ClawHub. Hub installs are staged with a heuristic scanner that the code itself calls 'NOT a sandbox' and fetch the repository's default branch with no pin or hash. MCP stdio servers are launched as the same user with the agent's entire environment, including every API key, so a malicious server inherits everything.

- **S L1:** Sources are user-chosen but unpinned (default branch for hub skills, whatever command the MCP config names). — [src/skills/hub/github-skill-client.ts:110](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/skills/hub/github-skill-client.ts#L110); [src/skills/hub/skill-security-scanner.ts:7](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/skills/hub/skill-security-scanner.ts#L7) (verified)
  - *To reach the next level:* No version pinning or integrity check for skills or MCP servers.
- **C L1:** Only hub skills get any pre-install check (a heuristic scan); MCP servers get none. — [src/skills/hub/skill-security-scanner.ts:7](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/skills/hub/skill-security-scanner.ts#L7) (verified)
  - *To reach the next level:* MCP servers and imported extensions are not verified.
- **D L2:** Nothing third-party is enabled out of the box; MCP servers and skills require an explicit add/install, and MCP config lives only in user-scope config.json. — [src/mcp/mcp-manager.ts:178-181](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/mcp/mcp-manager.ts#L178-L181) (verified)
  - *To reach the next level:* The add flow does not verify or display a pinned package and permission set, and workspace skill folders are loaded without consent.
- **B L0:** MCP stdio servers run as the same user with the full agent environment. — [src/mcp/mcp-client.ts:434](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/mcp/mcp-client.ts#L434); [AGENTS.md:519](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/AGENTS.md#L519) (verified)
  - *To reach the next level:* Extensions are not given a scrubbed environment or a sandbox.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.20 (high)

Secrets live in plaintext files (~/.atomic-agent/.env and config.json) written with owner-only permissions, and GitHub tokens and provider errors are scrubbed in some messages. Error reports to Sentry use a strict field allowlist, and PostHog analytics (on by default) is designed to carry no content. However traces are written unredacted by design, and every subprocess and MCP server receives every key in its environment, where the model can read them with a shell command.

- **S L1:** Secrets come from a 0600 .env loaded into process.env; masking exists for GitHub tokens and error reports but traces are explicitly unredacted. — [src/config/owner-only-file.ts:4](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/owner-only-file.ts#L4); [src/tracing/trace/trace-event.ts:17](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tracing/trace/trace-event.ts#L17); [src/error-reporting/error-scrubber.ts:2-7](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/error-reporting/error-scrubber.ts#L2-L7) (verified)
  - *To reach the next level:* No redaction on the trace/transcript path or before model-bound tool output.
- **C L1:** Error reporting is allowlist-scrubbed; traces, transcripts, model-bound messages and subprocess environments are not protected. — [src/error-reporting/error-scrubber.ts:2-7](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/error-reporting/error-scrubber.ts#L2-L7); [src/tracing/trace/trace-event.ts:17](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tracing/trace/trace-event.ts#L17); [src/sandbox/command-job.ts:141](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/sandbox/command-job.ts#L141) (verified)
  - *To reach the next level:* Logs and transcripts are not covered by redaction.
- **D L1:** Analytics is on by default but content-free; tracing to local NDJSON is on for CLI/TUI with no redaction. — [src/config/config-schema.ts:3141-3142](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/config-schema.ts#L3141-L3142); [src/tui/tui-command.ts:222](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tui/tui-command.ts#L222) (verified)
  - *To reach the next level:* Telemetry is not opt-in.
- **B L0:** Long-lived provider keys and GitHub PATs are reachable by every subprocess and the model. — [src/sandbox/command-job.ts:141](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/sandbox/command-job.ts#L141); [AGENTS.md:519](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/AGENTS.md#L519) (verified)
  - *To reach the next level:* Keys are neither scoped nor short-lived, and are exposed to all subprocesses.
- **Cap:** none

### C9 Audit & traceability — 0.50 (high)

With the CLI or TUI, every tool call is written to a per-session NDJSON trace (tool name, full arguments, status, timestamps) under ~/.atomic-agent/traces, and the session database records each prompted approval or denial with its category and time. The record does not say which person approved, auto-approvals are not logged as approvals, and the files sit where the agent's own process can rewrite them. If a trace write fails, tracing silently switches off for the session and the agent keeps going.

- **S L2:** Structured per-call trace events with arguments, status and timestamps, plus an approval ledger per tool call. — [src/tracing/trace/trace-event.ts:162-170](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tracing/trace/trace-event.ts#L162-L170); [src/approval/approval-ledger.ts:11-16](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/approval/approval-ledger.ts#L11-L16) (verified)
  - *To reach the next level:* No actor attribution (approver identity, requesting principal) on records.
- **C L2:** All tool calls go through the step executor and are traced; prompted approvals are recorded but auto-approvals are not. — [src/tracing/trace/trace-event.ts:162-170](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tracing/trace/trace-event.ts#L162-L170); [src/approval/approval-ledger.ts:11-16](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/approval/approval-ledger.ts#L11-L16) (verified)
  - *To reach the next level:* Auto-approved and refused-by-policy decisions are not recorded, and the headless task CLI defaults tracing off (src/cli/task-command.ts traceDefault: false).
- **D L2:** On by default for CLI/TUI in the state directory, outside the workspace, but writable by the agent process. — [src/tui/tui-command.ts:222](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tui/tui-command.ts#L222) (verified)
  - *To reach the next level:* The record is written by the same process the model drives.
- **B L2:** Events are appended synchronously per action; a write error is logged and tracing disabled while actions continue. — [src/tracing/trace/trace-sink.ts:206-216](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tracing/trace/trace-sink.ts#L206-L216) (verified)
  - *To reach the next level:* No fail-closed behaviour and no guarantee the trace survives a write error.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

A turn is capped at 25 steps per leg, 1,000 steps and two hours per task, with a 60-second default tool timeout and shell commands detached after ten minutes (hard-stopped after an hour). Stopping kills the whole process group of running commands. There is no token or cost ceiling, no rate limit on side-effecting tools, the model can ask for a shell call with no timeout, and scheduled and cron tasks it creates keep firing later.

- **S L2:** Step and wall-clock ceilings enforced in the loop, per-tool timeouts, and process-group kill on abort. — [src/config/config-schema.ts:2850-2868](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/config-schema.ts#L2850-L2868); [src/agent/agent-loop.ts:1640](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/agent/agent-loop.ts#L1640); [src/sandbox/command-job.ts:114](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/sandbox/command-job.ts#L114) (verified)
  - *To reach the next level:* No token or cost cap and no rate limits on side-effecting tools.
- **C L2:** Loop ceilings plus tool and shell timeouts; scheduled tasks start fresh budgets. — [src/config/config-schema.ts:2850-2868](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/config-schema.ts#L2850-L2868); searched `rg -n requireApproval` in `src/tools/memory src/tools/tasks` → 0 hits (memory writes and task scheduling are never gated) (verified)
  - *To reach the next level:* Scheduled tasks and fusion workers do not count against the parent task's budget.
- **D L1:** Sensible defaults exist, but the model can set timeoutMs 0 for unbounded shell calls and schedule new tasks. — [src/tools/os/shell-timeout.ts:34](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/tools/os/shell-timeout.ts#L34) (verified)
  - *To reach the next level:* The model can raise its own limits.
- **B L1:** Ceilings are large (2 h, 1,000 steps, unlimited spend) and model-created cron tasks persist after stop. — [src/config/config-schema.ts:2850-2868](https://github.com/atomicbot-ai/atomic-agent/blob/cce6262c135c97a15c020284a699dc6d164d34d3/src/config/config-schema.ts#L2850-L2868); searched `rg -n requireApproval` in `src/tools/memory src/tools/tasks` → 0 hits (memory writes and task scheduling are never gated) (verified)
  - *To reach the next level:* No tight time or spend ceiling; scheduled work continues after a stop.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: os.web.fetch / browser.* / MCP results enter context unmarked (src/tools/os/web-fetch.ts:112; rg untrusted in src/prompt: 0 hits) · [B] sensitive data/systems: workspace + home reads, full process.env with .env API keys and GITHUB_TOKEN (src/sandbox/command-job.ts:141, src/github/github-token.ts:11) · [C] state change / egress: os.http.request POST ungated by default (src/tools/os/http-request.ts:297) · Same default session? Yes

## Highest-impact improvements
1. Parse the full command with a shell parser and check each part before auto-allowing it. — C2 C L1→L2, +0.075 before caps (Playbook 5)
2. Default http.approvalMode to "writes" and gate browser.type/click form submission, so egress needs approval at level 1. — C5 B L0→L1, +0.050 before caps (Playbook 1)
3. Pass subprocesses and MCP servers a scrubbed baseline environment plus only the keys each one is configured with. — C7 B L0→L2, +0.100 before caps (Playbook 4)
4. Run shell commands and skill scripts under an OS sandbox profile (Seatbelt/Landlock) limiting writes to the workspace and denying network by default. — C4 S L0→L3, +0.225 before caps (Playbook 3)
5. Redact secret-shaped values from traces and tool output before they are written or sent to the model. — C8 C L1→L2, +0.075 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Shell-guard findings were established by reading the code paths, not by running the agent.
- The Tauri desktop host, TUI approval rendering, Telegram/Discord/swarm channels (off by default), Composio integration and fusion worker trace coverage were only skimmed.
- No text aimed at AI reviewers was found in the repository's Markdown files.
