# Defense-in-Depth Score: nanobot

**Repo:** https://github.com/HKUDS/nanobot · **Commit:** `b0d2382856a2857780181aefe09fd4ccb4fd8fd1` (0.3.5) · **Reviewed:** 2026-10-03
**What it is:** Ultra-lightweight self-hosted personal AI agent with tools, MCP and chat-app integrations
**Category:** AI Assistants
**Scored configuration:** Local pip/uv install running `nanobot webui` or `nanobot gateway` with the default ~/.nanobot/config.json: restrictToWorkspace=false, tools.exec.sandbox unset, exec/file/web/message/cron/spawn tools enabled, no chat channels enabled until configured.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 2.3 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L0 | L0 | 0.15 | C1-SELFESC | **0.15** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L1 | L2 | L0 | L0 | 0.23 | G1 | **0.23** | High |
| C4 | Code-execution isolation | L2 | L1 | L2 | L2 | 0.42 | G1 | **0.42** (alt) | High |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C7 | Third-party extensions | L0 | L1 | L0 | L0 | 0.07 | — | **0.07** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |


As shipped, nanobot gives the model an unsandboxed shell, unrestricted file access across the user's home directory, web access and the ability to message any configured chat, with no human approval for any of it. Its good controls (SSRF-safe web fetching, a bubblewrap/Seatbelt sandbox, workspace restriction, scrubbed subprocess environments, deny-by-default chat access with pairing) are either narrow or off by default. The dominant risk is prompt injection from web pages, email or chat turning into secret theft (config.json holds the API keys) and destructive commands with no one in the loop.

## Critical gaps
- In the default full-access mode the agent's file and shell tools can read and rewrite ~/.nanobot/config.json, so it can widen its own access settings that the self-configuration tool deliberately blocks, and it acts with the user's whole account. (ASI03, T3, LLM06; C1) — [nanobot/config/schema.py:402](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/config/schema.py#L402); [nanobot/agent/tools/filesystem.py:89-94](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/filesystem.py#L89-L94); [nanobot/agent/tools/self.py:92](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/self.py#L92)
- Shell commands run on the host as the user with no sandbox by default; the bwrap/Seatbelt sandbox is opt-in. (ASI05, T11, LLM05; C4) — [nanobot/agent/tools/shell.py:96-100](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L96-L100); [nanobot/agent/tools/shell.py:456](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L456)
- A hijacked agent can read secrets and exfiltrate them via web_fetch, exec or the message tool, and run destructive commands, all unattended. (ASI01, T6, LLM01; C5) — [nanobot/agent/tools/message.py:44-50](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/message.py#L44-L50); [nanobot/agent/tools/registry.py:196](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/registry.py#L196); [nanobot/config/loader.py:39](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/config/loader.py#L39)
- Python packages exposing a nanobot.tools entry point are loaded in-process automatically, and a bundled skill directs the model to install remote skills with npx --yes clawhub@latest. (ASI04, T17, LLM03; C7) — [nanobot/agent/tools/loader.py:74-79](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/loader.py#L74-L79); [nanobot/skills/clawhub/SKILL.md:30](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/skills/clawhub/SKILL.md#L30)

## Criterion details

### C1 Identity & least privilege — 0.15 (high)

nanobot runs every tool as the operating-system user who launched it. It does strip API keys and other variables from the shell tool's environment, which is a real narrowing, but in the default configuration the file and shell tools can read and write anywhere in the user's home directory, including SSH keys, cloud credential files and nanobot's own config.json with its provider keys and chat-bot tokens. Because file tools are unrestricted by default, the agent can also rewrite its own config.json (for example widening who may message it or re-enabling self-configuration), even though the built-in self-configuration tool deliberately blocks those fields. A hijacked agent therefore holds roughly the user's whole account.

- **S L1:** The only narrowing of ambient authority is environment scrubbing for shell subprocesses (HOME/LANG/TERM only); the agent otherwise acts with the full OS-user identity. — [nanobot/agent/tools/shell.py:827-833](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L827-L833) (verified)
  - *To reach the next level:* No dedicated or role-scoped identity: tools run as the launching user with access to every credential file in the home directory.
- **C L1:** The scrubbed environment applies to the exec tool, but file tools, CLI apps and MCP servers run with the same user and unrestricted filesystem reach by default. — [nanobot/agent/tools/shell.py:827-833](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L827-L833); [nanobot/agent/tools/filesystem.py:89-94](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/filesystem.py#L89-L94) (verified)
  - *To reach the next level:* Built-in file and app tools do not use any scoped identity; filesystem reach is only narrowed when restrict_to_workspace or a sandbox is enabled.
- **D L0:** The shipped default disables workspace restriction, so a fresh install runs with the user's full privileges. — [nanobot/config/schema.py:402](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/config/schema.py#L402) (verified)
  - *To reach the next level:* Least privilege (restrict_to_workspace or an exec sandbox) requires manual operator hardening.
- **B L0:** With default full access the agent can reach the user's entire account: home directory credential files, nanobot's config.json with provider keys and channel tokens, and any configured email/chat accounts. — [nanobot/config/loader.py:39](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/config/loader.py#L39); [nanobot/config/schema.py:402](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/config/schema.py#L402) (verified)
  - *To reach the next level:* No narrowing of what a hijacked agent can reach beyond the OS user's own permissions.
- **Cap:** C1-SELFESC — In the default full-access mode the write_file/exec tools can rewrite ~/.nanobot/config.json, changing the agent's own access settings (allowFrom, tools.my.allowSet, restrictToWorkspace), which MyTool deliberately blocks but the file tools do not.
- **Notes:** MyTool (self-configuration) is read-only by default and blocks restrict_to_workspace; the bypass is the generic file/shell path to the config file, which only the opt-in restriction or sandbox closes.

### C2 Approval gates — 0.00 (high)

There is no human approval step for any tool call. Shell commands, file writes and deletes, outbound chat messages with file attachments, scheduled jobs and MCP tool calls all execute as soon as the model requests them. The WebUI's 'restricted' versus 'full access' mode changes path checks, not approval. Mistakes or hijacked actions such as deleting files or messaging third parties happen with no human in the loop and largely cannot be undone.

- **S L0:** Tool calls are resolved, parameter-validated and executed directly; there is no approval mechanism of any kind. — searched `rg -n -i 'requires_approval|approve_tool|tool_approval|ask_permission|human_approval|needs_approval'` in `nanobot` → 0 hits (No approval or confirmation hook exists anywhere in the Python package; tool calls go straight from validation to execute().); [nanobot/agent/tools/registry.py:196](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/registry.py#L196) (verified)
  - *To reach the next level:* No per-call human approval exists for consequential tools.
- **C L0:** The most powerful tool (exec) is ungated, as is every other tool, including MCP and sub-agent tools. — [nanobot/agent/tools/registry.py:196](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/registry.py#L196); [nanobot/agent/tools/shell.py:96-100](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L96-L100) (verified)
  - *To reach the next level:* No gate exists for any tool path, including exec.
- **D L0:** Since no approval exists, it is not on by default. — searched `rg -n -i 'requires_approval|approve_tool|tool_approval|ask_permission|human_approval|needs_approval'` in `nanobot` → 0 hits (No approval or confirmation hook exists anywhere in the Python package; tool calls go straight from validation to execute().) (verified)
  - *To reach the next level:* Approval would need to exist and be on by default.
- **B L0:** Unapproved actions include arbitrary shell commands and outbound messages to any channel/chat ID with attachments; only the memory files have git-backed rollback. — [nanobot/agent/tools/message.py:44-50](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/message.py#L44-L50); [nanobot/agent/memory.py:89-90](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/memory.py#L89-L90) (verified)
  - *To reach the next level:* No checkpoints or undo for filesystem changes or external sends; irreversible actions are reachable.
- **Cap:** none

### C3 Tool & action scoping — 0.23 (high)

Tool input validation is uneven. The web fetch tool is well built: it blocks private, loopback and cloud-metadata addresses and re-checks every redirect. The file tools resolve symlinks and enforce workspace containment, and the shell tool has a deny-list and internal-URL check, but both of those only run when workspace restriction is turned on, and it is off by default. In the shipped configuration the shell tool takes arbitrary command strings with no filtering at all, and write, shell and network tools are all enabled.

- **S L1:** exec accepts raw shell strings and its regex deny-list only runs in restricted mode; web_fetch has allowlist-grade SSRF validation, but the flagship general tool is passthrough. — [nanobot/agent/tools/shell.py:446-447](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L446-L447); [nanobot/agent/tools/shell.py:216](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L216) (verified)
  - *To reach the next level:* No allowlist validation for the shell path; containment and filters are skipped unless restriction is enabled.
- **C L2:** Most built-in tools validate something (web_fetch SSRF with per-redirect checks, typed schemas via the registry, file path resolution), but MCP and entry-point plugin tools get only schema validation. — [nanobot/agent/tools/web.py:221-236](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/web.py#L221-L236); [nanobot/security/network.py:21](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/security/network.py#L21) (verified)
  - *To reach the next level:* Extension tools are not wrapped by a shared path/URL validation layer.
- **D L0:** Exec, file and web tools all default to enabled, with workspace restriction off. — [nanobot/agent/tools/shell.py:96-100](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L96-L100); [nanobot/agent/tools/filesystem.py:31](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/filesystem.py#L31); [nanobot/agent/tools/web.py:79](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/web.py#L79); [nanobot/config/schema.py:402](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/config/schema.py#L402) (verified)
  - *To reach the next level:* Dangerous tools are on by default and the default tool set includes write, exec and network.
- **B L0:** A misused exec or file tool can run any command or touch any file the user can reach, and contact any public host. — [nanobot/agent/tools/shell.py:446-447](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L446-L447); [nanobot/agent/tools/filesystem.py:89-94](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/filesystem.py#L89-L94) (verified)
  - *To reach the next level:* Tools are not scoped to the workspace or quantity-bounded by default.
- **Cap:** G1 — Path containment for file tools and the shell deny-list/internal-URL guard only run when restrictToWorkspace (default false) or a sandbox is enabled.

### C4 Code-execution isolation — 0.42 (high)

By default shell commands run directly on the host as the user, with no isolation. nanobot ships an optional bubblewrap (Linux) or Seatbelt (macOS) wrapper that limits filesystem writes to the workspace and hides the config directory, and the Docker image includes bubblewrap, but it is off unless the operator sets tools.exec.sandbox. Even when enabled, the sandbox leaves network access open, does not cover CLI apps or MCP servers, and on Windows silently runs unsandboxed with only a log warning. The criterion is credited for the opt-in sandbox, capped because it is off by default.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Default exec is a same-user subprocess (bash -c) with no isolation primitive. — [nanobot/agent/tools/shell.py:96-100](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L96-L100); [nanobot/agent/tools/shell.py:456](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L456) (verified)
    - *To reach the next level:* No isolation boundary in the default configuration.
  - **C L0:** No execution path is sandboxed by default. — [nanobot/agent/tools/shell.py:96-100](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L96-L100) (verified)
    - *To reach the next level:* The main exec tool is not sandboxed by default.
  - **D L0:** The sandbox backend defaults to the empty string (off). — [nanobot/agent/tools/shell.py:96-100](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L96-L100) (verified)
    - *To reach the next level:* Isolation is off by default.
  - **B L0:** Unsandboxed commands reach the whole home directory, including ~/.ssh and ~/.nanobot/config.json, with full network. — [nanobot/config/loader.py:39](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/config/loader.py#L39); [nanobot/agent/tools/shell.py:827-833](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L827-L833) (verified)
    - *To reach the next level:* Host-equivalent reach; nothing is mounted or networked narrowly.
- **opt-in bwrap/seatbelt exec sandbox (tools.exec.sandbox)** (alt; raw 0.42, cap G1 → 0.42) ← counted
  - **S L2:** bubblewrap with a mount namespace: system dirs read-only, workspace read-write, config parent masked by tmpfs; no --unshare-net, no seccomp or user-namespace hardening. — [nanobot/agent/tools/sandbox.py:84-93](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/sandbox.py#L84-L93); [nanobot/agent/tools/sandbox.py:205-206](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/sandbox.py#L205-L206) (verified)
    - *To reach the next level:* Network is not denied (no --unshare-net) and no seccomp/capability hardening, so it misses the hardened-sandbox anchor.
  - **C L1:** exec (and exec sessions via the same prepare path) are wrapped; the CLI-apps tool and MCP stdio servers are launched without the sandbox. — [nanobot/agent/tools/cli_apps.py:150-158](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/cli_apps.py#L150-L158); [nanobot/agent/tools/mcp.py:1075-1080](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/mcp.py#L1075-L1080) (verified)
    - *To reach the next level:* CLI apps and MCP stdio servers run on the host even with the sandbox enabled.
  - **D L2:** Enabling is a single config value; on Windows a configured sandbox logs a warning and runs unsandboxed. — [nanobot/agent/tools/shell.py:456-460](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L456-L460) (verified)
    - *To reach the next level:* No explicit operator flag or per-call approval governs unsandboxed execution; Windows falls back to host execution.
  - **B L2:** Inside bwrap the workspace is read-write, the config dir is masked and the env is scrubbed, but network egress is unrestricted and there are no CPU/PID limits. — [nanobot/agent/tools/sandbox.py:91-93](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/sandbox.py#L91-L93); [nanobot/agent/tools/shell.py:827-833](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L827-L833) (verified)
    - *To reach the next level:* Network egress is not disabled or allowlisted and no resource limits are applied.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.25 (high)

nanobot reads lots of content its owner did not write: web pages and search results, inbound email and chat messages, files, MCP tool results and other sessions' history. Web results and session history are wrapped with an 'untrusted data' banner and the system prompt asks the model not to follow embedded instructions, but nothing in code limits what a hijacked agent can then do. In the same session it can read the config file holding API keys, send data out through web requests, shell commands or chat messages, and run destructive commands, all without approval. Inbound chat channels do deny unknown senders by default and use operator-approved pairing, which keeps strangers from instructing the agent directly.

- **S L1:** Only spotlighting: an untrusted-content banner on web results and a notice on session history, plus a prompt instruction. — [nanobot/agent/tools/web.py:1253](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/web.py#L1253); [nanobot/templates/agent/_snippets/untrusted_content.md:1](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/templates/agent/_snippets/untrusted_content.md#L1) (verified)
  - *To reach the next level:* No capability is disabled or gated once untrusted content enters the session.
- **C L1:** Banners cover web_fetch/web_search and read_session; files, email bodies, MCP results and tool descriptions enter context unmarked. — [nanobot/agent/tools/web.py:1253](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/web.py#L1253); [nanobot/agent/tools/sessions.py:27](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/sessions.py#L27) (verified)
  - *To reach the next level:* Files, inbound email, MCP results and tool descriptions are not marked or limited.
- **D L2:** The banners are always applied and cannot be turned off by content, but they are a weak mechanism (capped one level above S). — [nanobot/agent/tools/web.py:1253](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/web.py#L1253) (verified)
  - *To reach the next level:* Mechanism is detection-only, so D cannot exceed one level above S.
- **B L0:** A hijacked default agent can read secrets (config.json) and exfiltrate them via web_fetch, exec or the message tool, and take irreversible actions via exec, with no human involved. — [nanobot/agent/tools/message.py:44-50](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/message.py#L44-L50); [nanobot/config/loader.py:39](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/config/loader.py#L39); [nanobot/agent/tools/registry.py:196](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/registry.py#L196) (verified)
  - *To reach the next level:* Neither exfiltration nor irreversible actions require approval.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.
- **Notes:** Channel access control (deny-by-default allowFrom with pairing, nanobot/channels/base.py:238-254) means public senders cannot directly instruct the agent, so C5-PUBLICTRIGGER is not applied.

### C6 Memory, context & configuration integrity — 0.05 (high)

Long-term memory (MEMORY.md), the persona files SOUL.md and USER.md, the project AGENTS.md and any 'always' workspace skills are injected into the system prompt on every turn. The model can write all of them with ordinary file tools, and a periodic 'Dream' job automatically consolidates conversation history into memory, so an injected instruction can persist and fire in later sessions. Memory is one store per workspace, shared across every channel and chat. There is a useful safety net: SOUL.md, USER.md and MEMORY.md are versioned in a local git store with /dream-log and /dream-restore, but AGENTS.md and skills are not.

- **S L0:** The model can write memory and bootstrap files freely and they are re-injected as trusted system-prompt context. — [nanobot/agent/context.py:128-130](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/context.py#L128-L130); [nanobot/agent/context.py:194-201](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/context.py#L194-L201) (verified)
  - *To reach the next level:* Memory writes are not gated, validated, or provenance-tagged.
- **C L0:** No store (memory, SOUL/USER, AGENTS.md, workspace skills, Dream summaries) has a write control. — [nanobot/config/schema.py:59](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/config/schema.py#L59); [nanobot/agent/skills.py:61](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/skills.py#L61) (verified)
  - *To reach the next level:* No memory or auto-loaded file path is controlled.
- **D L0:** A single MemoryStore per workspace is used for all channels and senders. — [nanobot/agent/context.py:98](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/context.py#L98) (verified)
  - *To reach the next level:* No per-user memory namespace.
- **B L1:** Poisoned memory persists across the user's sessions and can trigger tool use; git history allows rollback of three files only. — [nanobot/agent/memory.py:89-90](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/memory.py#L89-L90) (verified)
  - *To reach the next level:* AGENTS.md and skills are not versioned and poisoned memory is not limited to text output.
- **Cap:** none
- **Notes:** B would be L0 if the operator approves multiple chat users, since MEMORY.md is shared across them. No .env autoloading was found (rg load_dotenv: 0 hits).

### C7 Third-party extensions — 0.07 (high)

nanobot loads several kinds of third-party code. Agent Plugins are handled well: enabling one binds to a content fingerprint and any change disables it. Everything else is weaker. Python packages that declare a 'nanobot.tools' entry point are loaded automatically into the agent's own process, MCP servers are launched from whatever command the user configured (the built-in presets use unpinned @latest packages), and a bundled ClawHub skill tells the model to install skills from a public registry with 'npx --yes clawhub@latest', which the model can run through the ungated shell. A malicious extension runs with the user's full authority.

- **S L0:** Model-chosen installs of remote code are supported by a bundled skill (npx --yes clawhub@latest install) run through the ungated exec tool; MCP presets are unpinned @latest. — [nanobot/skills/clawhub/SKILL.md:30](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/skills/clawhub/SKILL.md#L30); [nanobot/webui/mcp_presets_api.py:149-150](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/webui/mcp_presets_api.py#L149-L150) (verified)
  - *To reach the next level:* Remote installs are not consent-gated and MCP sources are not pinned.
- **C L1:** Only Agent Plugins are integrity-bound (sha256 fingerprint checked on every load); MCP servers, entry-point tools and skills are not verified. — [nanobot/agent/plugins.py:416-433](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/plugins.py#L416-L433) (verified)
  - *To reach the next level:* MCP servers, entry-point tool plugins and skills are not pinned or integrity-checked.
- **D L0:** Any installed package with a nanobot.tools entry point is auto-registered, and skills installed into workspace/skills load without a separate consent step. — [nanobot/agent/tools/loader.py:74-79](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/loader.py#L74-L79) (verified)
  - *To reach the next level:* Extensions are enabled automatically once present.
- **B L0:** Entry-point plugins run in-process with the agent's credentials; MCP stdio servers run as the same user able to read config.json. — [nanobot/agent/tools/loader.py:74-79](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/loader.py#L74-L79); [nanobot/agent/tools/mcp.py:1075-1080](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/mcp.py#L1075-L1080) (verified)
  - *To reach the next level:* Extensions are not sandboxed or given scoped credentials.
- **Cap:** none
- **Notes:** MCP stdio env: nanobot passes cfg.env or None to the MCP SDK's StdioServerParameters; the SDK's default environment is a short allowlist (inferred from library behaviour), but the server can still read ~/.nanobot/config.json from disk.

### C8 Secrets & sensitive-data protection — 0.20 (high)

API keys, chat-bot tokens and email passwords live in ~/.nanobot/config.json, either in plaintext or as ${VAR} references, and nanobot does not enforce restrictive file permissions on it. The shell tool gets a scrubbed environment so keys are not inherited by commands, and API key fields are hidden from object representations. However, in the default full-access mode the model can simply read config.json, tool-call arguments are logged at INFO level without redaction, and transcripts are stored unencrypted. Telemetry (Langfuse) is only active when its keys are set.

- **S L1:** Secrets come from config/env refs with repr=False on key fields and a scrubbed subprocess env; there is no redaction of logs or model-bound content. — [nanobot/config/schema.py:202](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/config/schema.py#L202); [nanobot/agent/tools/shell.py:827-833](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L827-L833) (verified)
  - *To reach the next level:* No log/transcript redaction filters and no enforced restrictive permissions on stored credentials.
- **C L1:** Only the subprocess-environment path is protected; logs, transcripts and model-bound tool results are not. — [nanobot/agent/tools/shell.py:827-833](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L827-L833); [nanobot/agent/progress_hook.py:147-151](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/progress_hook.py#L147-L151) (verified)
  - *To reach the next level:* Logs and transcripts are not covered by redaction.
- **D L1:** Telemetry is opt-in, but tool-call arguments are logged at INFO by default with log_content=True and no redaction. — [nanobot/agent/progress_hook.py:32](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/progress_hook.py#L32); [nanobot/agent/progress_hook.py:151](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/progress_hook.py#L151) (verified)
  - *To reach the next level:* Default logging includes unredacted tool arguments.
- **B L0:** Long-lived provider keys, bot tokens and email credentials in config.json are readable by the model through file/shell tools in the default mode. — [nanobot/config/loader.py:39](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/config/loader.py#L39); [nanobot/agent/tools/filesystem.py:89-94](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/filesystem.py#L89-L94) (verified)
  - *To reach the next level:* Keys are long-lived and reachable by the model.
- **Cap:** none

### C9 Audit & traceability — 0.50 (high)

Every conversation is saved as a JSONL session transcript that includes tool calls, tool results and timestamps, stored outside the agent's workspace in a 0700 directory, and the runner writes a recovery checkpoint listing pending tool calls before they execute. Tool calls are also written to the human-readable log. The record is not tamper-evident, carries no actor or approver attribution, sub-agent tool calls are only summarised, and in the default full-access mode the agent's own tools could edit the files.

- **S L2:** Structured per-message transcript with tool_calls and timestamps. — [nanobot/session/manager.py:208-214](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/session/manager.py#L208-L214) (verified)
  - *To reach the next level:* No actor attribution (requesting principal, approver) or correlation IDs across sub-agents.
- **C L2:** Main-loop tool calls including MCP tools are recorded; sub-agent trajectories are not persisted in full. — [nanobot/agent/subagent.py:246](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/subagent.py#L246); [nanobot/agent/progress_hook.py:151](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/progress_hook.py#L151) (verified)
  - *To reach the next level:* Sub-agent tool calls and configuration/memory changes are not recorded in the same trail.
- **D L2:** On by default and stored outside the workspace (enforced), but the agent process and its unrestricted tools could alter it. — [nanobot/session/manager.py:456-463](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/session/manager.py#L456-L463) (verified)
  - *To reach the next level:* Records are written by the same process the model drives, and default file tools can reach them.
- **B L2:** A runtime checkpoint listing pending tool calls is persisted before tool execution, and the session is saved per turn. — [nanobot/agent/runner.py:506-514](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/runner.py#L506-L514) (verified)
  - *To reach the next level:* No guarantee the full trajectory is durably replayable per action, and failures do not block actions.
- **Cap:** none

### C10 Limits & kill switch — 0.45 (high)

Each agent run is capped at 200 tool iterations, shell commands time out after 60 seconds by default (the model can ask for up to 600), and MCP calls after 30 seconds. The /stop command cancels the session's tasks, sub-agents and background shell sessions and kills shell process groups. There is no wall-clock or cost limit, each sub-agent gets its own fresh 200-iteration budget, and scheduled jobs the agent created keep running after a stop.

- **S L2:** Iteration cap plus per-execution timeouts, enforced in code; halt cancels in-flight exec via process-group kill. — [nanobot/agent/runner.py:432](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/runner.py#L432); [nanobot/agent/tools/shell.py:97](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/shell.py#L97); searched `rg -n -i 'max_cost|cost_limit|spend_limit|budget_usd|max_spend|wall_clock|max_turn_seconds'` in `nanobot` → 4 hits (All 4 hits are an injectable clock in providers/oauth_model_catalog.py for model-catalog caching, not a run budget.) (verified)
  - *To reach the next level:* No wall-clock or token/cost cap and no rate limits on side-effecting tools.
- **C L2:** Top-level loop and tool timeouts are covered; sub-agents start a fresh budget rather than sharing the parent's. — [nanobot/agent/subagent.py:150-153](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/subagent.py#L150-L153); [nanobot/config/schema.py:374](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/config/schema.py#L374) (verified)
  - *To reach the next level:* Sub-agents and scheduled jobs do not count against the parent's budget.
- **D L2:** Defaults exist and MyTool cannot change max_iterations unless allowSet is enabled, but delegation resets the budget. — [nanobot/config/schema.py:130-131](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/config/schema.py#L130-L131); [nanobot/agent/tools/self.py:33](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/self.py#L33) (verified)
  - *To reach the next level:* The model can effectively reset its budget by spawning sub-agents.
- **B L1:** Stop cancels tasks, sub-agents and exec sessions and kills process groups, but model-created cron jobs keep firing and there is no spend ceiling. — [nanobot/agent/loop.py:906-907](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/loop.py#L906-L907); [nanobot/agent/tools/cron.py:68](https://github.com/HKUDS/nanobot/blob/b0d2382856a2857780181aefe09fd4ccb4fd8fd1/nanobot/agent/tools/cron.py#L68) (verified)
  - *To reach the next level:* Scheduled tasks survive a stop and there is no cost ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web_fetch results, inbound email/chat, MCP results (nanobot/agent/tools/web.py:1253) · [B] sensitive data/systems: provider keys and channel tokens in ~/.nanobot/config.json readable by file tools (nanobot/config/loader.py:39) · [C] state change / egress: ungated exec and message tools (nanobot/agent/tools/registry.py:196, nanobot/agent/tools/message.py:44-50) · Same default session? Yes

## Highest-impact improvements
1. Make restrictToWorkspace default to true so the existing path containment and shell guard apply out of the box. — C3 D L0→L2, +0.100 before caps (Playbook 3)
2. Enable the bwrap/Seatbelt exec sandbox by default where available, add --unshare-net (allowlisted egress) and fail closed on Windows. — C4 D L0→L2, +0.100 before caps (Playbook 3 step 1)
3. Add per-call human approval showing the exact command/recipient for exec, file writes outside the workspace, message sends and MCP calls. — C2 S L0→L3, +0.225 before caps (Playbook 5)
4. Once web/email/MCP content enters a session, force egress and state-changing tools through approval. — C5 S L1→L3, +0.150 before caps (Playbook 1)
5. Pin MCP preset versions and require explicit consent before skills or entry-point tool plugins are loaded; drop model-driven npx @latest installs. — C7 S L0→L2, +0.150 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Channel runtimes (Telegram, Discord, email, Feishu, Linear, etc.), the WebUI TypeScript client and the TUI were reviewed only for access control via the shared BaseChannel path, not individually.
- MCP stdio subprocess environment behaviour relies on the mcp SDK's default-environment handling (inferred, not verified in this repo).
- Docker/compose deployment (non-root user, cap_drop ALL, no-new-privileges) was footnoted, not scored; it would improve C4/C1 blast radius for container users.
- No reviewer-steering text was found (rg for auditor/ignore-instructions patterns outside tests/locales returned 0 hits).
