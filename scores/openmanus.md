# Defense-in-Depth Score: OpenManus

**Repo:** https://github.com/FoundationAgents/OpenManus · **Commit:** `3309bf4e416fb1c74b008f3e86494439a31bad53` · **Reviewed:** 2026-10-03
**What it is:** Open-source general agent (Manus alternative) with Python exec, browser and file tools
**Category:** AI Assistants
**Scored configuration:** `python main.py` (Manus agent) with config/config.toml copied from config.example.toml: sandbox off, default Browser Use MCP server on, no config/mcp.json.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents no · external communication yes

## Score: 1.2 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L0 | L1 | L0 | L0 | 0.07 | — | **0.07** | High |
| C4 | Code-execution isolation | L2 | L0 | L0 | L2 | 0.25 | G1 | **0.25** (alt) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L2 | 0.10 | C7-RCELOAD | **0.10** | Medium |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C9 | Audit & traceability | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | Medium |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | Medium |


OpenManus as shipped runs model-written Python directly on your machine as you, with your files, environment and network, and edits any file by absolute path, with no approval step anywhere. It also auto-launches an unpinned Browser Use package from PyPI that can drive your local Chrome. Any web page that hijacks the agent can read secrets (including the plaintext LLM key) and send them out, or destroy files, unattended. Run it only in a disposable VM or container that holds nothing you care about.

## Critical gaps
- Model-written Python runs via exec() in a same-user child process with full builtins, environment and network; the Docker sandbox is off by default and never used for python_execute. (ASI05, T11; C4) — [app/tool/python_execute.py:30](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L30); [app/tool/python_execute.py:61](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L61); [app/config.py:97](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/config.py#L97)
- The agent holds the launching user's entire authority with no authorization layer, including the user's Chrome via Browser Use. (ASI03, T3; C1) — [app/tool/python_execute.py:30](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L30); [README.md:98](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/README.md#L98)
- Untrusted web/MCP content enters context unmarked and a hijacked session can exfiltrate and destroy data unattended (C5-WORSTCASE). (ASI01, LLM01; C5) — [app/agent/toolcall.py:156-160](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/toolcall.py#L156-L160); [app/agent/manus.py:166-170](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L166-L170); [app/tool/python_execute.py:30](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L30)
- Browser Use is fetched unpinned from PyPI via uvx and executed at every start without consent (C7-RCELOAD). (ASI04, T17; C7) — [app/agent/manus.py:19-20](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L19-L20); [app/agent/manus.py:85-96](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L85-L96)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

OpenManus runs entirely as the operating-system user who launched it, with no identity of its own and no authorization check on any action. Model-written Python runs in a forked child of the agent process with the full user environment, and the file editor accepts any absolute path, so the agent can use whatever the user can: SSH keys, cloud credential files, the LLM API key in the config file. The default browser tool (Browser Use) is documented as attaching to the user's local Chrome automatically, which can extend that to the user's logged-in web sessions. A hijacked agent therefore holds the user's full authority.

- **S L0:** Ambient OS-user authority: model code runs via exec() in a forked child with full builtins and the inherited environment; no scoped identity exists. — [app/tool/python_execute.py:30](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L30); [app/tool/python_execute.py:61](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L61); [app/tool/python_execute.py:57-60](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L57-L60) (verified)
  - *To reach the next level:* No dedicated or narrowed identity; L1 needs at least a separate identity for the agent's actions.
- **C L0:** No tool passes through any authorization layer; tools act directly on the host (Path.write_text, exec, MCP calls). — [app/agent/toolcall.py:189](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/toolcall.py#L189); [app/tool/file_operators.py:57](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/file_operators.py#L57); searched `rg -n -i 'approv|confirm'` in `app main.py` → 0 hits (No approval or confirmation code anywhere in the agent.) (verified)
  - *To reach the next level:* No authorization check on any tool path; L1 needs the main tool path checked.
- **D L0:** The default install runs every tool with the launching user's full privilege; nothing narrows it. — [app/agent/manus.py:57-64](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L57-L64); [app/agent/manus.py:85-96](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L85-L96) (verified)
  - *To reach the next level:* Least privilege would require manual hardening; L1 needs a narrower default identity.
- **B L0:** A hijacked agent reaches everything the user can: home directory credential files, the plaintext LLM key, the network, and (per README) the user's own Chrome via Browser Use. — [app/tool/python_execute.py:30](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L30); [app/tool/str_replace_editor.py:171](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/str_replace_editor.py#L171); [README.md:98](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/README.md#L98); [app/config.py:220](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/config.py#L220) (verified)
  - *To reach the next level:* Nothing limits reach to one system; L1 needs authority narrower than the user's entire account.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no approval step anywhere in OpenManus. Python execution, file creation and editing, MCP tool calls (including Browser Use's browser_exec, which runs Python against the browser), and web actions all run as soon as the model asks for them. The only human-interaction tool, ask_human, is invoked at the model's discretion and gates nothing. Writes and code execution are immediate and mostly irreversible beyond the editor's in-memory undo.

- **S L0:** No approval mechanism exists; the executor runs any known tool directly. — [app/agent/toolcall.py:189](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/toolcall.py#L189); searched `rg -n -i 'approv|confirm'` in `app main.py` → 0 hits (No approval or confirmation code anywhere in the agent.) (verified)
  - *To reach the next level:* No human approval at all; L1 needs at least a blanket approval step.
- **C L0:** The most powerful tools (python_execute, MCP browser_exec) are ungated, as is every other tool. — [app/agent/manus.py:57-64](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L57-L64); [app/tool/mcp.py:28](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/mcp.py#L28); searched `rg -n -i 'approv|confirm'` in `app main.py` → 0 hits (No approval or confirmation code anywhere in the agent.) (verified)
  - *To reach the next level:* No tool is gated; L1 needs at least flagged tools to require approval.
- **D L0:** There is no approval to turn on; the default runs everything unattended. — searched `rg -n -i 'approv|confirm'` in `app main.py` → 0 hits (No approval or confirmation code anywhere in the agent.) (verified)
  - *To reach the next level:* Approval is absent; L1 needs it on by default.
- **B L0:** Unapproved actions include arbitrary Python on the host (delete files, send data anywhere) with no checkpoint; the editor's undo history is in-memory and covers only its own edits. — [app/tool/python_execute.py:30](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L30); [app/tool/str_replace_editor.py:101](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/str_replace_editor.py#L101) (verified)
  - *To reach the next level:* No rollback for code execution or external actions; L1 needs at least some actions to be reversible.
- **Cap:** none

### C3 Tool & action scoping — 0.07 (high)

The default tool set is about as broad as it gets: a tool that executes any Python code, a file editor that reads and writes any absolute path on the machine, and a browser tool that itself executes Python. The editor checks only that paths are absolute and that create does not overwrite, with no containment to the workspace, and its directory view is not confined either. Every tool is on by default.

- **S L0:** python_execute takes an arbitrary code string; the editor accepts any absolute path without containment. — [app/tool/python_execute.py:30](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L30); [app/tool/str_replace_editor.py:171](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/str_replace_editor.py#L171) (verified)
  - *To reach the next level:* Raw passthrough; L1 needs at least filtering of dangerous inputs.
- **C L1:** Only the editor validates anything (absolute path, existence, view_range bounds); python_execute and MCP tools pass arguments straight through. — [app/tool/str_replace_editor.py:171](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/str_replace_editor.py#L171); [app/tool/str_replace_editor.py:256-261](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/str_replace_editor.py#L256-L261); [app/tool/mcp.py:28](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/mcp.py#L28) (verified)
  - *To reach the next level:* Most tools do no validation; L2 needs most built-in tools to validate.
- **D L0:** Exec, write and browser/network tools are all enabled by default. — [app/agent/manus.py:57-64](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L57-L64); [app/agent/manus.py:85-96](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L85-L96) (verified)
  - *To reach the next level:* Everything enabled; L1 needs dangerous tools to be individually disableable (only Browser Use has an env-var off switch).
- **B L0:** A misused tool reaches the whole machine: any command via Python, any file via the editor, any host via the network. — [app/tool/python_execute.py:30](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L30); [app/tool/file_operators.py:57](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/file_operators.py#L57) (verified)
  - *To reach the next level:* No scoping; L1 needs at least minor limits on reach.
- **Cap:** none

### C4 Code-execution isolation — 0.25 (high)

Model-written Python runs with exec() in a forked child process of the agent, as the same user, with full builtins, the full environment and unrestricted network; the code even comments that it has 'safety restrictions', but only a timeout exists. Browser Use's browser_exec is a second Python execution path outside OpenManus' control. A Docker sandbox exists but is off by default, and even when enabled it is used only by the file editor, never by python_execute or MCP tools.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Same-user child process running exec() with full builtins; no isolation primitive. — [app/tool/python_execute.py:30](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L30); [app/tool/python_execute.py:57-60](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L57-L60); [app/tool/python_execute.py:61](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L61) (verified)
    - *To reach the next level:* No isolation; L1 needs at least filtering, L2 OS-level separation.
  - **C L0:** No execution path is sandboxed by default: python_execute, the editor, MCP stdio servers and browser_exec all run on the host. — [app/tool/python_execute.py:30](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L30); [app/tool/mcp.py:108](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/mcp.py#L108); [app/agent/manus.py:31](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L31) (verified)
    - *To reach the next level:* The main exec tool is unsandboxed; L1 needs at least the main exec tool sandboxed.
  - **D L0:** The only sandbox is off by default. — [app/config.py:97](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/config.py#L97) (verified)
    - *To reach the next level:* Sandbox off by default; L1 needs it on by default.
  - **B L0:** Executed code is host-equivalent: the user's home directory and credential files, the inherited environment, the plaintext LLM key and full network egress. — [app/tool/python_execute.py:30](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L30); [app/tool/python_execute.py:61](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L61); [app/config.py:220](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/config.py#L220) (verified)
    - *To reach the next level:* Host-equivalent reach; L1 needs at least no home-directory or credential access.
- **opt-in Docker sandbox (use_sandbox = true)** (alt; raw 0.25, cap G1 → 0.25) ← counted
  - **S L2:** Stock python:3.12-slim container with memory/CPU limits and network_mode none by default; root inside, default capabilities. — [app/sandbox/core/sandbox.py:65](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/sandbox/core/sandbox.py#L65); [app/config.py:98](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/config.py#L98) (verified)
    - *To reach the next level:* No hardening (non-root, dropped caps, seccomp, read-only root); L3 needs a hardened profile.
  - **C L0:** Only the editor's file operations use the sandbox; python_execute and MCP tools still run on the host. — [app/tool/str_replace_editor.py:110](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/str_replace_editor.py#L110); [app/tool/python_execute.py:30](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L30) (verified)
    - *To reach the next level:* The main exec tool bypasses the sandbox; L1 needs python_execute routed through it.
  - **D L0:** Opt-in via use_sandbox, default False. — [app/config.py:97](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/config.py#L97) (verified)
    - *To reach the next level:* Off by default; L1 needs it on by default.
  - **B L2:** Inside the container: no host mounts by the editor path, network off by default, memory/CPU limits, no secrets passed. — [app/sandbox/core/sandbox.py:65](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/sandbox/core/sandbox.py#L65); [app/tool/file_operators.py:105](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/file_operators.py#L105) (verified)
    - *To reach the next level:* No PID limit and the container runs as root; L3 needs full resource limits including PIDs.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.00 (high)

OpenManus feeds tool results, web content from Browser Use, MCP tool descriptions and MCP server instructions straight into the model's context; server instructions are even inserted as system messages. Nothing marks this content as untrusted, and nothing restricts what the agent can do after reading it. A web page that hijacks the agent can make it read local secrets with the editor or Python and send them anywhere over the network, or delete files, with no human involved.

- **S L0:** No structural limit on a hijacked agent; no taint tracking, approval or detection. — [app/agent/toolcall.py:156-160](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/toolcall.py#L156-L160); searched `rg -n -i 'untrusted|prompt injection|sanitiz'` in `app main.py` → 16 hits (Hits are MCP tool-name sanitizing (app/tool/mcp.py), a docstring in app/sandbox/__init__.py and command sanitizing inside the opt-in Docker terminal; none treats tool results or web content as untrusted.) (verified)
  - *To reach the next level:* Nothing limits a hijacked agent; L1 needs at least detection or spotlighting.
- **C L0:** Tool results enter as plain tool messages and MCP server instructions as system messages, with the same standing as the operator's instructions. — [app/agent/toolcall.py:156-160](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/toolcall.py#L156-L160); [app/agent/manus.py:166-170](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L166-L170); [app/tool/mcp.py:143](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/mcp.py#L143) (verified)
  - *To reach the next level:* Untrusted sources aren't distinguished; L1 needs at least one source handled.
- **D L0:** No control exists to enable. — searched `rg -n -i 'untrusted|prompt injection|sanitiz'` in `app main.py` → 16 hits (Hits are MCP tool-name sanitizing (app/tool/mcp.py), a docstring in app/sandbox/__init__.py and command sanitizing inside the opt-in Docker terminal; none treats tool results or web content as untrusted.) (verified)
  - *To reach the next level:* Off (absent); L1 needs a control on by default.
- **B L0:** A hijacked default session can read secrets (config.toml key, home files) and exfiltrate them via Python networking while also deleting or overwriting files, all unattended. — [app/tool/python_execute.py:30](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L30); [app/tool/file_operators.py:57](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/file_operators.py#L57); [app/agent/manus.py:85-96](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L85-L96); searched `rg -n -i 'approv|confirm'` in `app main.py` → 0 hits (No approval or confirmation code anywhere in the agent.) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions are both unattended; L1 needs at least one of them gated.
- **Cap:** C5-WORSTCASE — B is L0: a hijacked default session can leak data and take irreversible actions with no human involved.

### C6 Memory, context & configuration integrity — 0.10 (high)

OpenManus has no long-term memory store; conversation memory lives only in the running process. Its configuration (config/config.toml and config/mcp.json) is loaded from the installation directory rather than the current directory, so a cloned repository can't plant settings. However, the agent's own file editor and Python tool can write those files, and the next launch silently starts any MCP server listed in mcp.json and uses the LLM endpoint in config.toml. A single injected instruction can therefore persist as code that runs at every later start.

- **S L0:** The model can write config/mcp.json (via the unrestricted editor or Python), which adds stdio MCP servers that are launched without prompting on the next run. — [app/config.py:151](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/config.py#L151); [app/agent/manus.py:101-116](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L101-L116); [app/tool/str_replace_editor.py:171](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/str_replace_editor.py#L171); [app/tool/file_operators.py:57](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/file_operators.py#L57) (verified)
  - *To reach the next level:* No control on writes to security-relevant config; L1 needs at least logged writes and non-silent loading.
- **C L0:** Neither config file nor the files the agent writes are protected or checked on load. — [app/config.py:151](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/config.py#L151); [app/config.py:220](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/config.py#L220) (verified)
  - *To reach the next level:* No persistence path is controlled; L1 needs one controlled path.
- **D L1:** Single-user local install; config is read only from the install directory (not the working directory), but nothing stops the agent writing it. — [app/config.py:220](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/config.py#L220); [app/config.py:151](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/config.py#L151) (verified)
  - *To reach the next level:* The model can alter its own config; L2 needs enforced separation between agent-writable areas and configuration.
- **B L1:** A poisoned mcp.json persists across the user's sessions and launches arbitrary commands (tool use) at every start. — [app/agent/manus.py:101-116](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L101-L116); [app/tool/mcp.py:108](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/mcp.py#L108) (verified)
  - *To reach the next level:* Persistence can trigger code execution; L2 needs poisoned state to influence only text or gated actions.
- **Cap:** none

### C7 Third-party extensions — 0.10 (medium)

By default OpenManus launches Browser Use with `uvx browser-use --cli-mcp` at every start: an unpinned package fetched from PyPI and run without any explicit consent step, which exposes a tool that executes Python in the browser harness. Additional MCP servers come from config/mcp.json with no pinning or verification. Extension processes are separate, and the MCP client library passes them a reduced environment by default, but they run as the same user with full filesystem and network access.

- **S L0:** Default-on `uvx browser-use` resolves whatever version is current and executes it; no pin, hash or signature. — [app/agent/manus.py:19-20](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L19-L20); [app/agent/manus.py:85-96](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L85-L96); searched `rg -n 'sha256|hashlib|verify'` in `app/tool/mcp.py app/agent/manus.py app/config.py` → 0 hits (No pinning, hashing or verification of MCP servers or the browser-use package.) (verified)
  - *To reach the next level:* Unverified, unpinned remote code runs automatically; L1 needs user-chosen sources (L2 pinned versions).
- **C L0:** Neither the built-in Browser Use launch nor config-listed MCP servers are verified. — [app/agent/manus.py:101-116](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L101-L116); searched `rg -n 'sha256|hashlib|verify'` in `app/tool/mcp.py app/agent/manus.py app/config.py` → 0 hits (No pinning, hashing or verification of MCP servers or the browser-use package.) (verified)
  - *To reach the next level:* No extension type is verified; L1 needs one verified type.
- **D L0:** Browser Use is enabled automatically unless OPENMANUS_DISABLE_BROWSER_USE is set. — [app/agent/manus.py:85-96](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L85-L96); [README.md:114](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/README.md#L114) (verified)
  - *To reach the next level:* Extension enabled by default; L1 needs at least a consent prompt on first use.
- **B L2:** MCP servers run as separate stdio processes; OpenManus passes env=None (or only Browser Use variables), so the mcp SDK supplies its default minimal environment (inferred from mcp library behaviour), but the process keeps the user's filesystem and network. — [app/tool/mcp.py:108](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/mcp.py#L108); [app/agent/manus.py:37-38](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L37-L38) (inferred)
  - *To reach the next level:* Not sandboxed per extension; L3 needs per-extension sandboxing with scoped credentials.
- **Cap:** C7-RCELOAD — By default the agent runs `uvx browser-use`, an unpinned remote package, without consent.

### C8 Secrets & sensitive-data protection — 0.00 (high)

The LLM API key sits in plaintext in config/config.toml inside the install directory, which the agent's own editor and Python tool can read, and the forked Python child inherits the agent's full environment. There is no redaction anywhere: the default log file at DEBUG level records model thoughts, tool arguments and full tool results. There is no telemetry, which is good, but nothing keeps credentials away from the model or from code it runs.

- **S L0:** Plaintext API key in a config file readable by the model's tools; no masking in any path. — [app/config.py:22](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/config.py#L22); [config/config.example.toml:5](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/config/config.example.toml#L5); [app/config.py:220](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/config.py#L220); searched `rg -n -i 'redact|SecretStr'` in `app` → 0 hits (No redaction or secret-typing anywhere.) (verified)
  - *To reach the next level:* No masking at all; L1 needs masking in at least one path.
- **C L0:** No path (logs, model-bound tool results, subprocess environment) is protected. — searched `rg -n -i 'redact|SecretStr'` in `app` → 0 hits (No redaction or secret-typing anywhere.); [app/agent/toolcall.py:151-153](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/toolcall.py#L151-L153) (verified)
  - *To reach the next level:* No path protected; L1 needs one.
- **D L0:** Verbose payload logging (tool arguments and full results) to a file is on by default; no telemetry. — [app/logger.py:12-25](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/logger.py#L12-L25); [app/agent/toolcall.py:151-153](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/toolcall.py#L151-L153); searched `rg -n -i 'sentry|posthog|telemetry'` in `app main.py` → 1 hits (Only hit sets ANONYMIZED_TELEMETRY=false inside the opt-in Daytona sandbox; no telemetry SDK in the agent.) (verified)
  - *To reach the next level:* Verbose payload logging is on by default; L1 needs content logging to be opt-in or redacted.
- **B L0:** A long-lived LLM provider key (and any cloud or Browser Use keys in the environment) is reachable by model-written code and every subprocess. — [app/tool/python_execute.py:61](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L61); [app/config.py:220](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/config.py#L220); [app/agent/manus.py:21-22](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L21-L22) (verified)
  - *To reach the next level:* Long-lived keys reachable by the model; L1 needs keys kept from model-run code.
- **Cap:** none

### C9 Audit & traceability — 0.33 (medium)

OpenManus writes a loguru text log at DEBUG level to logs/ under the install directory by default, recording each step, each tool name, and each tool's full result. Arguments are logged only for the first tool call of each step, so parallel calls lose their arguments, and records are unstructured text with no actor or correlation fields. The log directory is reachable by the agent's own editor and Python tool, so a hijacked agent can rewrite or delete its own record.

- **S L1:** Unstructured text log lines; arguments recorded only for tool_calls[0]. — [app/agent/toolcall.py:89](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/toolcall.py#L89); [app/agent/toolcall.py:151-153](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/toolcall.py#L151-L153); [app/logger.py:12-25](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/logger.py#L12-L25) (verified)
  - *To reach the next level:* Not a structured record of every call's arguments; L2 needs every call's arguments, status and timestamps.
- **C L2:** All tools, including MCP tools, go through the same execute_tool path that logs name and result. — [app/agent/toolcall.py:189](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/toolcall.py#L189); [app/agent/toolcall.py:151-153](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/toolcall.py#L151-L153) (verified)
  - *To reach the next level:* Approvals, sub-agent attribution and config changes are not recorded; L3 needs all calls plus approvals/denials.
- **D L1:** On by default, written to PROJECT_ROOT/logs, which the unrestricted editor and Python tool can modify. — [app/logger.py:25](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/logger.py#L25); [app/tool/str_replace_editor.py:171](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/str_replace_editor.py#L171) (verified)
  - *To reach the next level:* The agent can alter its own logs; L2 needs storage the agent's tools can't reach.
- **B L1:** Logging is best-effort via loguru; failures are not surfaced and nothing blocks actions on a missing record. — [app/logger.py:25](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/logger.py#L25); [app/agent/toolcall.py:151-153](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/toolcall.py#L151-L153) (inferred)
  - *To reach the next level:* No guaranteed per-action durability or replayable trajectory; L2 needs surfaced errors and per-action flushing.
- **Cap:** none

### C10 Limits & kill switch — 0.33 (medium)

The Manus agent stops after 20 steps, and Python execution has a 5-second default timeout. But the model can pass a larger timeout argument, which the tool accepts even though it isn't in the schema; there is no token budget by default, no wall-clock limit, and MCP/browser calls have no timeout. Terminating a timed-out Python run kills only the direct child, so anything it spawned keeps running.

- **S L2:** Step cap (20) plus a per-execution timeout on python_execute, enforced in code. — [app/agent/manus.py:51](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/manus.py#L51); [app/agent/base.py:136-138](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/base.py#L136-L138); [app/tool/python_execute.py:39-43](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L39-L43); [app/config.py:24-27](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/config.py#L24-L27); searched `rg -n -i 'wall|deadline|time\.monotonic|max_cost|budget'` in `app/agent` → 0 hits (No wall-clock or cost budget in the agent loop.) (verified)
  - *To reach the next level:* No wall-clock or token/cost cap by default; L3 needs step, time and cost caps plus rate limits.
- **C L1:** The step cap covers the loop; only python_execute has a timeout, and MCP/browser calls are unbounded. — [app/agent/base.py:136-138](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/agent/base.py#L136-L138); [app/tool/mcp.py:28](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/mcp.py#L28); [app/tool/python_execute.py:39-43](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L39-L43) (verified)
  - *To reach the next level:* Tool timeouts don't cover MCP tools and are model-overridable; L2 needs loop plus tool timeouts.
- **D L1:** Defaults exist (20 steps, 5 s) but the model can raise the Python timeout by passing a timeout argument, since tool inputs are splatted into execute(). — [app/tool/python_execute.py:39-43](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L39-L43); [app/tool/tool_collection.py:32](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/tool_collection.py#L32) (verified)
  - *To reach the next level:* Model can raise its limits; L2 needs defaults the model cannot change.
- **B L1:** proc.terminate() signals only the direct child, so processes started by model code outlive the timeout and the agent; no spend ceiling. — [app/tool/python_execute.py:68-70](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/tool/python_execute.py#L68-L70); [app/config.py:24-27](https://github.com/FoundationAgents/OpenManus/blob/3309bf4e416fb1c74b008f3e86494439a31bad53/app/config.py#L24-L27) (inferred)
  - *To reach the next level:* Stopping can leave spawned work running; L2 needs moderate ceilings and nothing left running.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Browser Use MCP tool results and server instructions (app/agent/manus.py:85-96, 166-170) · [B] sensitive data/systems: plaintext LLM key and user files readable by the editor and Python (app/config.py:220, app/tool/str_replace_editor.py:171) · [C] state change / egress: python_execute exec() with network and file writes (app/tool/python_execute.py:30, app/tool/file_operators.py:57) · Same default session? Yes

## Highest-impact improvements
1. Route python_execute (and the editor) through the Docker sandbox and turn it on by default with network off. — C4 C L0→L3, +0.225 before caps (Playbook 3)
2. Add a per-call human approval showing the exact code/path/content before python_execute, file writes and MCP calls. — C2 S L0→L3, +0.225 before caps (Playbook 5)
3. Pin Browser Use to a version and hash, and require explicit opt-in instead of auto-launching it. — C7 S L0→L2, +0.150 before caps (Playbook 3)
4. Confine the editor to the workspace with resolved-path containment and harden its directory view. — C3 S L0→L3, +0.225 before caps (Playbook 3)
5. Ignore undeclared tool arguments (e.g. timeout) and add a wall-clock and token budget on by default. — C10 D L1→L2, +0.050 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Browser Use (fetched by uvx at runtime) was not reviewed; its behaviour, including attaching to the user's Chrome, is taken from OpenManus' README and is not verified in code.
- MCP stdio environment scrubbing (C7 B) is inferred from the mcp Python SDK's default behaviour, not verified in this repo.
- Only the primary `python main.py` mode was scored; run_flow.py, run_mcp.py, sandbox_main.py (Daytona), the A2A server and the SWE/browser/data-analysis agents were not scored.
- No reviewer-steering or prompt-injection text aimed at auditors was found in the repository.
