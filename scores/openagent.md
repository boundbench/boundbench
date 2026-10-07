# Defense-in-Depth Score: OpenAgent

**Repo:** https://github.com/the-open-agent/openagent · **Commit:** `c9a8fca73702a55b4c3bd171461a02fc8f8cef0b` · **Reviewed:** 2026-10-04
**What it is:** Self-hosted personal AI assistant (Go web app, single binary) with RAG, agent loops, shell, local-file, office, browser and desktop tools, MCP and skills.
**Category:** AI Assistants
**Scored configuration:** Single-binary install with shipped defaults (SQLite fallback, local sign-in), built-in store with Tools=All and Skills=All, chatting as the global admin.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents no · external communication opt-in

## Score: 1.3 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L0 | L0 | 0.07 | — | **0.07** | High |
| C2 | Approval gates | L0 | L0 | L0 | L1 | 0.05 | C2-SELFAPPROVE | **0.05** | High |
| C3 | Tool & action scoping | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | C7-RCELOAD | **0.00** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |


OpenAgent ships with every built-in tool enabled, including an unsandboxed host shell that inherits the server's full environment, local file read/write, desktop GUI control and browser automation, and none of them require human approval. A casbin-based permission engine exists in the repo but is never called from the tool path. Authentication and access control in the default install are not locked down. A hijacked session (for example via a fetched web page) can exfiltrate data and take irreversible actions on the host with nobody in the loop.

## Critical gaps
- No approval gate or isolation: the default store exposes an unsandboxed host shell with full environment to the model, and the permission engine is never wired in. (ASI05, T11; C4) — [tool/shell.go:527-528](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L527-L528); [tool/shell.go:743-744](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L743-L744); [audit/audit.go:49-50](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/audit/audit.go#L49-L50)
- A hijacked default session can exfiltrate and take irreversible host actions unattended (C5-WORSTCASE). (ASI01, LLM01, T6; C5) — [controllers/message_answer.go:227](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/controllers/message_answer.go#L227); [object/init.go:128-129](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/init.go#L128-L129)
- A default-enabled skill tells the model to npm-install and fetch third-party skills, which the ungated shell carries out without consent (C7-RCELOAD). (ASI04, T17; C7) — [skills/clawhub/SKILL.md:28](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/skills/clawhub/SKILL.md#L28); [object/init.go:128](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/init.go#L128)
- local_file_move's only confirmation is a confirmed=true argument the model supplies itself (C2-SELFAPPROVE). (ASI09, T10; C2) — [tool/local_file.go:737-738](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/local_file.go#L737-L738)

## Criterion details

### C1 Identity & least privilege — 0.07 (high)

OpenAgent runs every tool as the operating-system user that launched the server, and the shell tool passes the server's full environment to each command. The one real authorization check is that high-risk tool types (shell, files, GUI, browser) are only exposed when the requester is the global admin; store admins and ordinary users get them filtered out. Enforcement of that check does not cover every entry point, and authentication in the default install is not locked down.

- **S L0:** Tools run with the server process's ambient OS-user authority and full environment; there is no dedicated or narrowed identity. — [tool/shell.go:743-744](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L743-L744); [tool/shell.go:527-528](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L527-L528) (verified)
  - *To reach the next level:* Give tool execution a dedicated low-privilege identity instead of the server's OS user and environment.
- **C L1:** The chat path gates high-risk tools on the global-admin principal, but enforcement does not cover every entry point, and every subprocess gets the full environment. — [controllers/message_answer.go:65](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/controllers/message_answer.go#L65); [controllers/message_answer.go:218-219](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/controllers/message_answer.go#L218-L219) (verified)
  - *To reach the next level:* Apply the high-risk-tool principal check consistently and stop passing os.Environ to subprocesses.
- **D L0:** Authentication in the default install is not locked down, including for the admin role that unlocks the shell. (verified)
  - *To reach the next level:* Harden default authentication and network exposure.
- **B L0:** A hijacked admin session controls a host shell as the server's OS user, with its files, credentials and network. — [tool/shell.go:527-528](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L527-L528); [object/init.go:128-129](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/init.go#L128-L129) (verified)
  - *To reach the next level:* Confine tool authority to a scoped, revocable identity rather than the host user account.
- **Cap:** none

### C2 Approval gates — 0.05 (high)

There is no human approval anywhere in the tool path: the agent loop executes whatever tool calls the model returns, including shell commands, file writes, GUI automation and browser actions. A tri-state allow/ask/deny policy engine (guard/) and a tool-policy table exist, but the host never builds or consults the guard, and the audit code itself notes it is not yet wired in. The system prompt actively tells the model not to refuse. File writes and moves through local_file are snapshotted so they can be reverted, but shell actions are not.

- **S L0:** No approval step exists; tool calls go straight from the model to execution. — searched `rg -n NewCasbinGuard` in `object controllers model tool routers main.go` → 0 hits (The casbin guard engine in guard/ is never constructed by the host; only guard/ and its tests reference it.); [audit/audit.go:49-50](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/audit/audit.go#L49-L50); searched `rg -n -i 'RequestApproval|requireApproval'` in `model object controllers tool` → 0 hits (No approval call anywhere in the host's tool path.) (verified)
  - *To reach the next level:* Wire the guard into callMcpTool with an Approver that shows the exact call to a human.
- **C L0:** The most powerful tool (shell) and every other tool reach execution without any gate. — [model/mcp.go:193](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/model/mcp.go#L193); [model/mcp.go:357](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/model/mcp.go#L357) (verified)
  - *To reach the next level:* Route every builtin and MCP tool call through one gate before execution.
- **D L0:** No gate is on by default; the guard's ask default never runs because the guard is never constructed. — [audit/audit.go:49-50](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/audit/audit.go#L49-L50) (verified)
  - *To reach the next level:* Turn the guard on by default with ask for exec/write/network categories.
- **B L1:** local_file_write and local_file_move snapshot files for rollback, but shell, GUI, browser and MCP actions are irreversible and unbounded. — [object/snapshot_tool.go:32-35](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/snapshot_tool.go#L32-L35); [tool/shell.go:527-528](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L527-L528) (verified)
  - *To reach the next level:* Add rollback or previews beyond local_file writes, especially for shell-driven changes.
- **Cap:** C2-SELFAPPROVE — The only confirmation step in the tool set, local_file_move's confirmed argument, is a value the model itself supplies.
- **Notes:** local_file_move requires confirmed=true (tool/local_file.go:737-738), but that flag is set by the model, so it is self-approval, not a human gate; levels unchanged and the cap is non-binding.

### C3 Tool & action scoping — 0.15 (high)

Tool scoping is mixed. web_fetch has a real SSRF guard that resolves the host and rejects non-public addresses, re-checking on redirects. But the default tool set is everything, including a raw shell string, and local_file only requires that paths be absolute, so any file the OS user can reach is in scope. An optional shellCommandAllowlist config limits the first word of shell commands, but it is empty by default.

- **S L1:** web_fetch checks resolved IPs against public ranges, but shell is a raw command string and local_file accepts any absolute path; the shell allowlist is a first-token match behind a metacharacter denylist. — [tool/web_fetch.go:86-91](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/web_fetch.go#L86-L91); [tool/local_file.go:195-203](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/local_file.go#L195-L203); [object/tool_audit.go:85-89](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/tool_audit.go#L85-L89) (verified)
  - *To reach the next level:* Replace raw shell with narrow tools and constrain local_file to resolved roots.
- **C L1:** Only web_fetch (and the opt-in shell allowlist) validate arguments meaningfully; shell, local_file, office, GUI and MCP tools pass arguments through. — [tool/web_fetch.go:86-91](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/web_fetch.go#L86-L91); [tool/local_file.go:195-203](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/local_file.go#L195-L203) (verified)
  - *To reach the next level:* Validate arguments for every built-in tool and wrap MCP tools in a shared validation layer.
- **D L0:** The built-in store enables all tools by default, including shell, file write, GUI and browser. — [object/init.go:128-129](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/init.go#L128-L129) (verified)
  - *To reach the next level:* Ship a read-only default tool set and require explicit enabling of exec/write tools.
- **B L0:** A misused shell tool can run any command on the host with no quantity or scope bounds. — [tool/shell.go:527-528](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L527-L528) (verified)
  - *To reach the next level:* Scope tools to a workspace with quantity bounds.
- **Cap:** none
- **Notes:** D here rates least-agency posture (all tools on by default), not whether validation is opt-in; web_fetch's SSRF check is always on, so G1 does not apply.

### C4 Code-execution isolation — 0.00 (high)

Shell commands run as an ordinary subprocess of the server on the host, using sh -c and the server's full environment. Nothing in the tool, object or model packages sets up a container, OS sandbox or restricted user; the only 'sandbox' references disable Chrome's own sandbox for the browser tools. The Docker image does run as a non-root user, but that user is given passwordless sudo, and the single-binary install that the README leads with has no isolation at all.

- **S L0:** Commands run as a same-user host subprocess with no isolation primitive. — [tool/shell.go:527-528](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L527-L528); searched `rg -n -i 'sandbox|seccomp|landlock|nsjail|firejail|bwrap'` in `tool object model mcp controllers` → 2 hits (Both hits are chromedp no-sandbox flags that disable Chrome's own sandbox; no isolation for shell or tools.) (verified)
  - *To reach the next level:* Run shell and code tools in a hardened container or OS sandbox.
- **C L0:** No execution path is sandboxed: shell, background PTY sessions, MCP stdio servers and the browser (with no-sandbox) all run on the host. — searched `rg -n -i 'sandbox|seccomp|landlock|nsjail|firejail|bwrap'` in `tool object model mcp controllers` → 2 hits (Both hits are chromedp no-sandbox flags that disable Chrome's own sandbox; no isolation for shell or tools.); [tool/browser_use.go:215](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/browser_use.go#L215); [mcp/client.go:123-127](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/mcp/client.go#L123-L127) (verified)
  - *To reach the next level:* Route every exec path through one sandbox backend.
- **D L0:** There is no sandbox to enable. — searched `rg -n -i 'sandbox|seccomp|landlock|nsjail|firejail|bwrap'` in `tool object model mcp controllers` → 2 hits (Both hits are chromedp no-sandbox flags that disable Chrome's own sandbox; no isolation for shell or tools.) (verified)
  - *To reach the next level:* Ship a sandbox on by default.
- **B L0:** Commands reach the whole host as the server user, with the server's environment, the SQLite DB holding provider keys, and full network. — [tool/shell.go:743-744](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L743-L744); [object/provider.go:47](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/provider.go#L47) (verified)
  - *To reach the next level:* Limit reachable filesystem, secrets and network from executed code.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Web pages, search results, uploaded documents, knowledge-base passages and MCP results all enter the model's context with no marking or provenance, and nothing in code changes what the agent may do after reading them. The system prompt added when tools are present tells the model to act immediately and never refuse. In the default configuration a successful injection can use the shell or web_fetch to send data out and take irreversible actions on the host without any human seeing it first. Opt-in chat pipes widen this further when configured.

- **S L0:** Nothing limits a hijacked agent; the only related text is a prompt pushing the model to comply. — [controllers/message_answer.go:227](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/controllers/message_answer.go#L227) (verified)
  - *To reach the next level:* Disable or approval-gate egress and state-changing tools once untrusted content enters a session.
- **C L0:** Tool results, fetched pages and knowledge are appended as ordinary messages with no source distinction. — [model/mcp.go:357](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/model/mcp.go#L357); searched `rg -n -i 'untrusted|prompt injection|injection'` in `model/mcp.go controllers/message_answer.go` → 0 hits (No provenance or injection handling in the agent loop.) (verified)
  - *To reach the next level:* Tag every untrusted source and apply the same restriction to all of them.
- **D L0:** No control exists to be on by default. — [controllers/message_answer.go:227](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/controllers/message_answer.go#L227) (verified)
  - *To reach the next level:* Ship a Rule-of-Two restriction on by default.
- **B L0:** A hijacked default session can exfiltrate via web_fetch or shell and run destructive shell commands, unattended. — [object/init.go:128-129](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/init.go#L128-L129); [tool/shell.go:527-528](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L527-L528) (verified)
  - *To reach the next level:* Make sessions that read untrusted content lose egress or require approval for it.
- **Cap:** C5-WORSTCASE — B is L0: leak plus irreversible action, unattended, in the default configuration.

### C6 Memory, context & configuration integrity — 0.20 (high)

Persistent context comes from per-chat history, store knowledge bases, an opt-in experience library, and skills. All skills are injected into every default store's prompt, and skill files are re-synced into the database from a skills folder next to the binary at startup. There is no memory-write tool, and the LLM-driven experience review that writes skills automatically is off by default. However, the ungated shell can write that skills folder or the SQLite database directly, so an injection can plant instructions that every later session and user receives.

- **S L1:** No dedicated memory-write tool and auto skill-writing is opt-in, but skills load silently as high-priority prompt content and nothing validates writes made via the shell. — [object/experience_review.go:182](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/experience_review.go#L182); [controllers/message_answer.go:240-246](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/controllers/message_answer.go#L240-L246); [object/init.go:336](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/init.go#L336) (verified)
  - *To reach the next level:* Gate writes to skills and memory and present loaded skills as data with provenance.
- **C L1:** Only the experience-review write path has a control (being opt-in); skills, the skills folder and the database are uncontrolled. — [object/experience_review.go:182](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/experience_review.go#L182); [object/init.go:336](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/init.go#L336) (verified)
  - *To reach the next level:* Cover skills, the skills folder, knowledge and experience stores with the same write control.
- **D L1:** Chats are per user, but skills are global across stores and the knowledge base is shared by everyone using a store. — [object/init.go:128](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/init.go#L128) (verified)
  - *To reach the next level:* Namespace persistent context per user by default and enforce it in queries.
- **B L0:** A skill planted in the skills folder or DB persists across sessions and users and is phrased as instructions to use tools such as the shell. — [object/init.go:336](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/init.go#L336); [controllers/message_answer.go:240-246](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/controllers/message_answer.go#L240-L246) (verified)
  - *To reach the next level:* Require human review before new persistent context reaches other sessions.
- **Cap:** none

### C7 Third-party extensions — 0.00 (high)

MCP servers are added by the global admin as raw stdio commands or URLs with no pinning or integrity check, and launch as the server's OS user. Skills (all enabled by default) are prompt files, but the bundled clawhub skill tells the model to npm-install a CLI and fetch new skills from clawhub.com on the fly; with the ungated shell, the model can install and run arbitrary third-party packages without anyone approving it.

- **S L0:** Model-chosen package installs are possible and encouraged by a default-enabled skill; MCP commands are unpinned. — [skills/clawhub/SKILL.md:28](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/skills/clawhub/SKILL.md#L28); [mcp/client.go:123-127](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/mcp/client.go#L123-L127) (verified)
  - *To reach the next level:* Pin and verify extensions and block model-initiated package installs.
- **C L0:** Neither MCP servers, skills nor shell-installed packages are verified. — [mcp/client.go:123-127](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/mcp/client.go#L123-L127); [skills/clawhub/SKILL.md:28](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/skills/clawhub/SKILL.md#L28) (verified)
  - *To reach the next level:* Verify every extension type.
- **D L0:** All bundled skills are enabled by default and the model can install packages without consent. — [object/init.go:128](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/init.go#L128); [skills/clawhub/SKILL.md:28](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/skills/clawhub/SKILL.md#L28) (verified)
  - *To reach the next level:* Enable no third-party skills by default and require operator consent for installs.
- **B L0:** Installed packages and stdio MCP servers run as the same OS user as the server with access to its files and the DB. — [mcp/client.go:123-127](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/mcp/client.go#L123-L127); [tool/shell.go:743-744](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L743-L744) (verified)
  - *To reach the next level:* Confine extensions in separate sandboxed processes with scoped credentials.
- **Cap:** C7-RCELOAD — By default the model can install and execute remote packages (npm i -g, clawhub skill fetch) through the ungated shell without consent.

### C8 Secrets & sensitive-data protection — 0.20 (high)

Provider API keys and tool secrets are stored in plain text in the database (SQLite next to the binary by default), and masked as *** in API responses. Tool-audit records redact sensitive JSON fields, but the agent loop prints every tool result and knowledge passage to stdout unredacted, and the shell tool hands the server's full environment to every command. The shipped configuration's default database credentials are also not locked down. No third-party telemetry was found.

- **S L1:** Secrets come from env and plaintext DB columns; redaction exists on the tool-audit path only. — [object/provider.go:47](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/provider.go#L47); [object/tool_audit.go:130](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/tool_audit.go#L130) (verified)
  - *To reach the next level:* Encrypt stored secrets and redact before stdout logs and model-bound messages.
- **C L1:** Only tool-audit records are redacted; stdout logs, transcripts and subprocess environments are not. — [object/tool_audit.go:130](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/tool_audit.go#L130); [model/mcp.go:416](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/model/mcp.go#L416); [tool/shell.go:743-744](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L743-L744) (verified)
  - *To reach the next level:* Extend redaction to logs, transcripts and subprocess environments.
- **D L1:** No telemetry, but verbose tool-result logging to stdout is on by default and unredacted. — [model/mcp.go:416](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/model/mcp.go#L416) (verified)
  - *To reach the next level:* Make payload logging opt-in and redaction always on.
- **B L0:** Long-lived provider keys in the DB and every env secret are reachable by the model through the shell. — [tool/shell.go:743-744](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L743-L744); [object/provider.go:47](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/provider.go#L47) (verified)
  - *To reach the next level:* Keep long-lived keys out of reach of model-driven subprocesses.
- **Cap:** none

### C9 Audit & traceability — 0.40 (high)

Tool calls are recorded in three places: the chat transcript stores each call's name, arguments and result; high-risk built-in tools also write a database record with redacted arguments and the user; and a JSONL audit file per session records every tool call's name, outcome and duration (but only the argument length). All of these live where the server process, and therefore the shell tool, can edit them. The JSONL writer drops events when its queue is full and swallows write errors, and the transcript is saved only when the answer finishes.

- **S L2:** Structured per-call records exist (transcript with arguments, DB record with user for high-risk tools, JSONL with outcome and timing). — [controllers/message_answer.go:519](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/controllers/message_answer.go#L519); [object/tool_audit.go:137](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/tool_audit.go#L137); [model/mcp.go:308](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/model/mcp.go#L308) (verified)
  - *To reach the next level:* Add approver and requesting-principal attribution and correlation IDs to every record.
- **C L2:** Built-in and MCP calls reach the transcript and JSONL; the admin GetAnswerWithTool path skips the DB audit wrapper and no approvals exist to record. — [object/message_tool.go:47](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/message_tool.go#L47); [model/mcp.go:308](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/model/mcp.go#L308) (verified)
  - *To reach the next level:* Record every path, including GetAnswerWithTool, and record approvals and denials.
- **D L1:** Logging is on by default, but the DB and audit directory sit next to the binary where the shell tool can rewrite them. — [audit/audit.go:120](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/audit/audit.go#L120); [tool/shell.go:527-528](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L527-L528) (verified)
  - *To reach the next level:* Write audit records from a component the model's tools cannot modify.
- **B L1:** JSONL events are dropped under load and write errors are swallowed; the transcript is flushed only at the end of the answer. — [audit/audit.go:92-95](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/audit/audit.go#L92-L95); [controllers/message_answer.go:519](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/controllers/message_answer.go#L519) (verified)
  - *To reach the next level:* Flush records per action and surface write failures.
- **Cap:** none

### C10 Limits & kill switch — 0.20 (high)

The agent loop keeps calling the model and running tools for as long as the model returns tool calls; there is no round, time, or token cap. Individual shell commands time out after 30 seconds by default, but the model can raise that to 300 seconds and start background shell sessions that live until 30 minutes of idleness. Cancelling an answer stops output being written, but tool calls run under a background context, so in-flight commands keep going. A per-user rate limit applies only to signed-out users and defaults to 10,000 messages per 15 minutes.

- **S L1:** Only per-execution timeouts exist; there is no iteration, wall-clock or cost cap on the loop. — [model/mcp.go:193](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/model/mcp.go#L193); searched `rg -n -i 'maxIteration|maxRounds|maxSteps|maxToolRounds'` in `model controllers object` → 0 hits (No round or step cap on the agent loop.); [tool/shell.go:38-44](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L38-L44) (verified)
  - *To reach the next level:* Add an iteration cap plus wall-clock and token limits enforced in code.
- **C L1:** Timeouts apply to individual shell and fetch calls; the loop and background sessions are unbounded. — [tool/shell.go:38-44](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L38-L44); [model/mcp.go:296](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/model/mcp.go#L296) (verified)
  - *To reach the next level:* Bound the top-level loop and count background sessions against it.
- **D L1:** Defaults are very large or absent, and the model can raise the shell timeout to 300 seconds per call. — [tool/shell.go:38-44](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/tool/shell.go#L38-L44); [object/init.go:113](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/object/init.go#L113); [controllers/message_answer.go:276-283](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/controllers/message_answer.go#L276-L283) (verified)
  - *To reach the next level:* Set sensible loop limits by default that the model cannot raise.
- **B L0:** A runaway can loop indefinitely, and cancel leaves in-flight tools and background shells running. — [model/mcp.go:193](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/model/mcp.go#L193); [model/mcp.go:296](https://github.com/the-open-agent/openagent/blob/c9a8fca73702a55b4c3bd171461a02fc8f8cef0b/model/mcp.go#L296) (verified)
  - *To reach the next level:* Cap runs and cancel in-flight tool execution on stop.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web_fetch/web_search/browser tools enabled by default (object/init.go:128-129; tool/web_fetch.go) · [B] sensitive data/systems: host files and full process env via shell (tool/shell.go:743-744), provider keys in DB (object/provider.go:47) · [C] state change / egress: ungated shell and local_file_write (model/mcp.go:357; tool/shell.go:527-528) · Same default session? Yes

## Highest-impact improvements
1. Wire the existing guard into callMcpTool with ask as the default for exec/write/network categories and a UI approver that shows the exact call. — C2 S L0→L3, +0.225 before caps (Playbook 5)
2. Harden default authentication and network exposure of the install. — C1 D L0→L2, +0.100 before caps (Playbook 4)
3. Ship the built-in store with a read-only tool set; make shell, file write, GUI and browser explicit opt-ins. — C3 D L0→L3, +0.150 before caps (Playbook 3)
4. Add a hard cap on agent rounds plus a wall-clock limit, and pass the job's context into tool execution so cancel kills in-flight commands. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
5. Scrub the shell subprocess environment to a minimal allowlist. — C8 C L1→L2, +0.075 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Environment inheritance of stdio MCP servers depends on the go-mcp library's WithStdioClientOptionEnv behaviour, which was not inspected (not vendored); C7 B is rated on the same-user process alone.
- The React frontend (web/), the 50+ bundled SKILL.md files beyond clawhub, pipe transports other than WeixinClaw, and the BPMN workflow/task scheduler were only spot-checked.
- No text aimed at AI reviewers was found in the repository (README, CLAUDE.md, SECURITY.md checked).
