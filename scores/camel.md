# Defense-in-Depth Score: CAMEL

**Repo:** https://github.com/camel-ai/camel · **Commit:** `0106b76830c707effe48cd3da384dda38cbed92e` (0.2.91a7) · **Reviewed:** 2026-10-03
**What it is:** Multi-agent framework / research on agent scaling
**Category:** Agent Frameworks
**Scored configuration:** Python library: ChatAgent and bundled toolkits (TerminalToolkit, CodeExecutionToolkit, FileToolkit, WebFetch, Gmail, MCPToolkit) at their constructor defaults, no hardening code added.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents yes · external communication yes

## Score: 1.9 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L2 | L1 | L0 | L0 | 0.23 | G1 | **0.23** (alt) | High |
| C3 | Tool & action scoping | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C4 | Code-execution isolation | L4 | L1 | L0 | L2 | 0.47 | G1 | **0.47** (alt) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L3 | 0.20 | — | **0.20** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L1 | 0.23 | — | **0.23** | High |
| C8 | Secrets & sensitive-data protection | L1 | L0 | L1 | L0 | 0.12 | — | **0.12** | High |
| C9 | Audit & traceability | L1 | L1 | L0 | L0 | 0.15 | G1 | **0.15** | High |
| C10 | Limits & kill switch | L2 | L1 | L0 | L0 | 0.23 | G1 | **0.23** | High |


CAMEL gives agents powerful bundled tools (shell, code execution, browser, email, MCP) but almost no guardrails on by default. Model-chosen shell commands and code run on the host as your user with your full environment, the agent loop has no approval gate or step/time limit, and nothing limits what injected web or email content can make an agent do. The one default prompt (host code execution) asks y/N without showing the code at the default log level. Treat any CAMEL agent with execution or messaging toolkits as fully trusted with your machine and accounts unless you add isolation (E2B/Docker) and approvals yourself.

## Critical gaps
- Shell and code-execution subprocesses inherit the full host environment, so any credential in the developer's shell is reachable by model-written commands. (ASI03, T3; C1) — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:486](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L486); [camel/interpreters/subprocess_interpreter.py:467](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/interpreters/subprocess_interpreter.py#L467)
- The most powerful bundled tool (TerminalToolkit.shell_exec) runs model-chosen shell commands with no approval by default. (ASI02, ASI09, T10; C2) — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:148](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L148)
- Default code and shell execution run on the host as the user with the full process environment, so a hijacked agent's code reaches host files, network and credentials. (ASI05, T11; C4) — [camel/toolkits/code_execution.py:65](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/code_execution.py#L65); [camel/interpreters/subprocess_interpreter.py:467](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/interpreters/subprocess_interpreter.py#L467); [camel/toolkits/terminal_toolkit/terminal_toolkit.py:486](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L486)
- Injected content can drive both exfiltration and irreversible actions unattended: nothing tracks untrusted input or gates egress/state-changing tools. (ASI01, LLM01, T6; C5) — [camel/toolkits/gmail_toolkit.py:540](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/gmail_toolkit.py#L540); [camel/toolkits/gmail_toolkit.py:76](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/gmail_toolkit.py#L76); [camel/agents/chat_agent.py:4126](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/agents/chat_agent.py#L4126)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

CAMEL has no identity or authorization layer of its own. Each toolkit builds its own client from environment variables or local token files, and the ChatAgent runs whatever tool the model names with no per-request authorization check. The shell and code-execution tools pass the full process environment to every subprocess, so any cloud, Git or API credential the developer has in their shell is available to model-written commands. The bundled Gmail toolkit requests read, send and modify scopes in one token and exposes all of them together.

- **S L0:** Toolkits use ambient credentials: subprocesses inherit the full os.environ and the Gmail toolkit asks for read+send+modify scopes in a single grant. — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:486](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L486); [camel/toolkits/gmail_toolkit.py:26-34](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/gmail_toolkit.py#L26-L34) (verified)
  - *To reach the next level:* No dedicated or role-scoped identity; credentials are whatever the operator's environment holds.
- **C L0:** Every toolkit constructs its own privileged client; the agent loop calls tools directly with no authorization layer. — [camel/agents/chat_agent.py:4126](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/agents/chat_agent.py#L4126); [camel/interpreters/subprocess_interpreter.py:467](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/interpreters/subprocess_interpreter.py#L467) (verified)
  - *To reach the next level:* No shared authorization layer that tool calls pass through.
- **D L0:** Default constructors run with the operator's full authority; narrowing requires the developer to scrub the environment and pick narrower scopes in their own code. — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:486](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L486); [camel/toolkits/gmail_toolkit.py:26-34](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/gmail_toolkit.py#L26-L34) (verified)
  - *To reach the next level:* No narrower default identity is shipped.
- **B L0:** A hijacked agent with the shell or code tool reaches every credential in the developer's environment (cloud CLIs, tokens), and with Gmail can read, send and trash mail. — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:486](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L486); [camel/interpreters/subprocess_interpreter.py:467](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/interpreters/subprocess_interpreter.py#L467); [camel/toolkits/gmail_toolkit.py:713](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/gmail_toolkit.py#L713) (verified)
  - *To reach the next level:* Nothing limits credential reach to one system or read-only operations.
- **Cap:** none

### C2 Approval gates — 0.23 (high)

The ChatAgent executes every registered tool immediately; there is no framework-level approval step. The bundled code-execution toolkit asks 'Running code? [y/N]' by default when it runs on the host, but the code itself is only written to an INFO log line that the default WARNING log level hides, so the person approving does not see what will run. The terminal toolkit, the most powerful bundled tool, has an approval callback that shows the exact command, but it is off by default and does not cover its own file-writing tool. An LLM-based risk judge (LLMGuardRuntime) exists but an LLM judge is not human approval.

- **default configuration** (default; raw 0.17, cap C2-POWERBYPASS → 0.17)
  - **S L1:** Host code-execution interpreters prompt per call, but the code is emitted via logger.info while the library configures WARNING, so the approver sees only a bare y/N prompt. — [camel/interpreters/subprocess_interpreter.py:379-390](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/interpreters/subprocess_interpreter.py#L379-L390); [camel/logger.py:30](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/logger.py#L30); [camel/toolkits/code_execution.py:79-83](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/code_execution.py#L79-L83) (verified)
    - *To reach the next level:* The approver must see the exact code or command being approved.
  - **C L0:** The terminal toolkit's shell_exec is ungated by default and the agent loop calls all other tools without any gate. — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:148](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L148); [camel/toolkits/terminal_toolkit/terminal_toolkit.py:566-570](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L566-L570); [camel/agents/chat_agent.py:4126](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/agents/chat_agent.py#L4126) (verified)
    - *To reach the next level:* Every consequential tool, starting with the shell, must traverse the gate.
  - **D L2:** The interpreter prompt is on by default for host sandboxes and fails closed without a TTY, but a constructor flag silently disables it. — [camel/toolkits/code_execution.py:79-83](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/code_execution.py#L79-L83); [camel/interpreters/subprocess_interpreter.py:78](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/interpreters/subprocess_interpreter.py#L78) (verified)
    - *To reach the next level:* Disabling should need a loudly named operator flag, and the gate should be on for the shell tool too.
  - **B L0:** Ungated shell, email-sending and trash tools can take irreversible actions; there is no checkpoint or undo. — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:1560](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L1560); [camel/toolkits/gmail_toolkit.py:76](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/gmail_toolkit.py#L76) (verified)
    - *To reach the next level:* No rollback or previews for consequential actions.
- **opt-in TerminalToolkit require_approval callback (_default_console_approval)** (alt; raw 0.23, cap G1 → 0.23) ← counted
  - **S L2:** Per-call console approval shows the exact sanitized command and denies on non-interactive stdin. — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:69-87](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L69-L87) (verified)
    - *To reach the next level:* No risk tiers or argument-level allow/deny policy.
  - **C L1:** Only shell_exec and shell_write_to_process call the gate; shell_write_content_to_file and every other toolkit bypass it. — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:804](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L804); [camel/toolkits/terminal_toolkit/terminal_toolkit.py:1390](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L1390) (verified)
    - *To reach the next level:* All mutating tools, including file writes and other toolkits, must traverse the gate.
  - **D L0:** The callback defaults to None, i.e. no approval. — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:148](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L148) (verified)
    - *To reach the next level:* Approval should be on by default.
  - **B L0:** Same irreversible reach as the default: file writes and other toolkits stay ungated. — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:1390](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L1390) (verified)
    - *To reach the next level:* No rollback or previews.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C3 Tool & action scoping — 0.25 (high)

Argument validation is thin. The terminal toolkit's safe mode is a denylist of command names (rm, sudo, dd and so on), and its enforcement, including path containment for cd and file writes, is not a strict boundary. The file toolkit accepts absolute paths anywhere on disk, and the web-fetch tool accepts any http(s) URL and follows redirects with no block on internal addresses. FunctionTool does not validate model arguments against the schema before calling the function.

- **S L1:** The main validation is a command-name denylist plus path containment checks whose enforcement is not a strict boundary. — [camel/toolkits/terminal_toolkit/utils.py:61-72](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/utils.py#L61-L72) (verified)
  - *To reach the next level:* Allowlist validation in code: robust path containment, host allowlists with internal-address blocking.
- **C L1:** Only the terminal toolkit validates; FileToolkit takes absolute paths unchecked and WebFetch only checks the scheme. — [camel/toolkits/file_toolkit.py:96-102](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/file_toolkit.py#L96-L102); [camel/toolkits/web_fetch_toolkit.py:139](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/web_fetch_toolkit.py#L139); [camel/toolkits/function_tool.py:625-646](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/function_tool.py#L625-L646) (verified)
  - *To reach the next level:* Most built-in tools should validate their inputs.
- **D L2:** Developers pick toolkits, but each toolkit exposes all its tools, including write and exec, by default. — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:1559-1566](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L1559-L1566); [camel/toolkits/gmail_toolkit.py:1811-1836](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/gmail_toolkit.py#L1811-L1836) (verified)
  - *To reach the next level:* A read-only default tool set with write/exec requiring explicit enabling.
- **B L0:** General-purpose shell and arbitrary URL fetch reach the whole machine and any host. — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:838-848](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L838-L848); [camel/toolkits/web_fetch_toolkit.py:32](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/web_fetch_toolkit.py#L32) (verified)
  - *To reach the next level:* Tools scoped to a workspace with bounded operations.
- **Cap:** none

### C4 Code-execution isolation — 0.47 (high)

By default, model-written code and commands run directly on the host as the developer's user. CodeExecutionToolkit defaults to a 'subprocess' sandbox that is just a local subprocess, and TerminalToolkit defaults to a local backend whose only protection is a command-name denylist. Both pass the full environment to the child process. Stronger options exist: a Docker interpreter (stock container, non-root user, no other hardening) and an E2B remote sandbox, which is a real boundary but is opt-in and only covers the code-execution toolkit.

- **default configuration** (default; raw 0.25 → 0.25)
  - **S L1:** Default execution is a same-user host subprocess; the terminal toolkit adds only denylist filtering. — [camel/toolkits/code_execution.py:65](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/code_execution.py#L65); [camel/interpreters/subprocess_interpreter.py:469-476](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/interpreters/subprocess_interpreter.py#L469-L476); [camel/toolkits/terminal_toolkit/utils.py:243-249](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/utils.py#L243-L249) (verified)
    - *To reach the next level:* An OS-level boundary (container, low-privilege user) around default execution.
  - **C L1:** The denylist covers TerminalToolkit.shell_exec only; CodeExecutionToolkit and InternalPythonInterpreter.execute_command run on the host unfiltered. — [camel/interpreters/subprocess_interpreter.py:438-476](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/interpreters/subprocess_interpreter.py#L438-L476); [camel/interpreters/internal_python_interpreter.py:643-662](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/interpreters/internal_python_interpreter.py#L643-L662) (verified)
    - *To reach the next level:* Most exec paths should go through the boundary.
  - **D L2:** safe_mode is on by default but a constructor argument disables it without warning. — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:143](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L143); [camel/toolkits/terminal_toolkit/utils.py:243-244](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/utils.py#L243-L244) (verified)
    - *To reach the next level:* Disabling should need an explicit loudly named flag, with per-call approval for unsandboxed execution.
  - **B L0:** Executed code runs on the host with the full environment (credentials) and unrestricted network. — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:486](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L486); [camel/interpreters/subprocess_interpreter.py:467](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/interpreters/subprocess_interpreter.py#L467) (verified)
    - *To reach the next level:* Isolate from host files and secrets.
- **opt-in E2B remote sandbox (CodeExecutionToolkit(sandbox='e2b'))** (alt; raw 0.47, cap G1 → 0.47) ← counted
  - **S L4:** E2B runs code in a remote ephemeral sandbox service, killed on interpreter teardown. — [camel/interpreters/e2b_interpreter.py:81](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/interpreters/e2b_interpreter.py#L81); [camel/interpreters/e2b_interpreter.py:241](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/interpreters/e2b_interpreter.py#L241) (verified)
  - **C L1:** Covers only the code-execution toolkit; TerminalToolkit and other tools still run on the host. — [camel/toolkits/code_execution.py:119-122](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/code_execution.py#L119-L122); [camel/toolkits/terminal_toolkit/terminal_toolkit.py:136-148](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L136-L148) (verified)
    - *To reach the next level:* Every exec path, including the terminal toolkit, routed to the sandbox.
  - **D L0:** Opt-in; the default sandbox is 'subprocess'. — [camel/toolkits/code_execution.py:65](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/code_execution.py#L65) (verified)
    - *To reach the next level:* Make the isolated backend the default.
  - **B L2:** No host files or host env in the remote sandbox, but network egress is not restricted by CAMEL. — [camel/interpreters/e2b_interpreter.py:81](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/interpreters/e2b_interpreter.py#L81) (verified)
    - *To reach the next level:* Egress off or allowlisted, with resource limits.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.00 (high)

CAMEL does nothing to limit what injected content can make an agent do. Web pages, emails, browser content and MCP tool results enter the conversation as ordinary tool messages with no provenance or untrusted marking, and nothing disables or gates egress or state-changing tools once such content has been read. A single agent built from bundled toolkits can read untrusted email or web content, hold the developer's credentials, and send email or run shell commands without a human in the loop.

- **S L0:** No taint tracking, provenance, quarantine, or approval tied to untrusted content. — searched `rg -n -i 'prompt.injection|untrusted|taint|provenance|quarantin'` in `camel` → 2 hits (Both hits are prose (microsandbox docstring, an agent prompt); no control.) (verified)
  - *To reach the next level:* At least approval for dangerous tools after untrusted content is read.
- **C L0:** Tool results are recorded as ordinary function messages regardless of source. — [camel/agents/chat_agent.py:4253-4263](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/agents/chat_agent.py#L4253-L4263) (verified)
  - *To reach the next level:* Distinguish untrusted sources from principal input.
- **D L0:** No control exists to be on by default. — searched `rg -n -i 'prompt.injection|untrusted|taint|provenance|quarantin'` in `camel` → 2 hits (Same search; no control.) (verified)
  - *To reach the next level:* Ship a default limit on untrusted-content sessions.
- **B L0:** A hijacked agent can exfiltrate (web fetch, gmail send) and act irreversibly (shell, trash) with no human involved. — [camel/toolkits/gmail_toolkit.py:76](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/gmail_toolkit.py#L76); [camel/toolkits/gmail_toolkit.py:540](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/gmail_toolkit.py#L540); [camel/toolkits/terminal_toolkit/terminal_toolkit.py:1560](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L1560) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions must need human approval.
- **Cap:** C5-WORSTCASE — B is L0: leak plus irreversible action unattended in default configuration.

### C6 Memory, context & configuration integrity — 0.20 (high)

The default agent memory is an in-memory chat history tied to one agent object, so poisoning normally dies with the session. When persistence is used there are no write controls: the optional MemoryToolkit lets the model load arbitrary JSON records (any role) into its own memory and save memory to any path, and vector or JSON stores accept writes without validation or provenance. The optional SkillToolkit auto-discovers SKILL.md instructions from the working directory and gives them priority over the user's own skills, but skills are text only and cannot enable tools or change security settings.

- **S L0:** MemoryToolkit lets the model replace its memory with arbitrary records and write memory files anywhere; no validation or provenance on memory writes. — [camel/toolkits/memory_toolkit.py:66-93](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/memory_toolkit.py#L66-L93); [camel/toolkits/memory_toolkit.py:54-64](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/memory_toolkit.py#L54-L64) (verified)
  - *To reach the next level:* Memory entries should carry provenance and be presented as data.
- **C L0:** No memory store or auto-loaded file path is controlled; repo-scope skills load silently ahead of user scope. — [camel/toolkits/skill_toolkit.py:200-206](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/skill_toolkit.py#L200-L206) (verified)
  - *To reach the next level:* Control the main memory store.
- **D L1:** Default memory is a per-agent in-memory store; there is no tenant/namespace isolation in persistent stores beyond what the developer builds. — [camel/agents/chat_agent.py:567-571](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/agents/chat_agent.py#L567-L571); [camel/memories/blocks/chat_history_block.py:51](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/memories/blocks/chat_history_block.py#L51) (verified)
  - *To reach the next level:* Per-user/session namespaces enforced in queries for persistent stores.
- **B L3:** By default memory is session-scoped and in-process; auto-summaries (on by default) are written as markdown into a per-session folder and only reloaded by explicit calls, so they are easy to inspect and purge. — [camel/memories/blocks/chat_history_block.py:51](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/memories/blocks/chat_history_block.py#L51); [camel/agents/chat_agent.py:510](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/agents/chat_agent.py#L510); [camel/agents/chat_agent.py:1060](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/agents/chat_agent.py#L1060) (verified)
  - *To reach the next level:* Persistent memory only after human review, with rollback.
- **Cap:** none

### C7 Third-party extensions — 0.23 (high)

CAMEL loads third-party code mainly through MCP servers the developer lists in a config, and through Hugging Face models in a few toolkits. Nothing is pinned, hashed, or re-approved when it changes; MCP stdio servers are launched as given by the config. MCP servers run as separate processes and, unless the config passes env, the MCP SDK gives them a reduced environment (library behaviour, inferred). The Jina reranker toolkit's local mode loads its model with trust_remote_code=True, running remote model code in-process, though that mode is opt-in.

- **S L1:** MCP servers come from developer-chosen config with no version pinning or integrity checks. — [camel/utils/mcp_client.py:536-542](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/utils/mcp_client.py#L536-L542); [camel/toolkits/jina_reranker_toolkit.py:81-85](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/jina_reranker_toolkit.py#L81-L85) (verified)
  - *To reach the next level:* Pin versions of extensions.
- **C L0:** No extension type is verified. — searched `rg -n 'sha256|checksum|verify_signature|integrity'` in `camel/utils/mcp_client.py camel/toolkits/mcp_toolkit.py` → 0 hits (No integrity checks on MCP servers.) (verified)
  - *To reach the next level:* Verify at least one extension type.
- **D L2:** MCP servers are added only by explicit developer config (config_path/config_dict), not from the workspace, but nothing shows what will run. — [camel/toolkits/mcp_toolkit.py:333-337](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/mcp_toolkit.py#L333-L337) (verified)
  - *To reach the next level:* Show the exact package, command and permissions when adding an extension.
- **B L1:** MCP servers run as separate same-user processes; HF remote code in the Jina local mode runs in-process. — [camel/utils/mcp_client.py:539](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/utils/mcp_client.py#L539); [camel/toolkits/jina_reranker_toolkit.py:81-85](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/jina_reranker_toolkit.py#L81-L85) (verified)
  - *To reach the next level:* Separate process with a scrubbed environment for every extension type.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.12 (high)

API keys are read from environment variables and are not masked anywhere. At INFO level the agent logs every model request in full with no secret redaction (the default WARNING level hides it, but one environment variable turns it on). Shell and code subprocesses inherit the whole environment, so a model-written 'env' command reveals every key. Telemetry integrations (AgentOps, Langfuse, Traceroot) are opt-in via environment variables, and the Gmail token file is written with 0600 permissions.

- **S L1:** Secrets come from env vars; no type-level masking or redaction helpers exist. — searched `rg -n 'SecretStr|redact|mask_secret|keyring'` in `camel` → 5 hits (All hits are Anthropic 'redacted_thinking' block handling, not secret masking.) (verified)
  - *To reach the next level:* Type-level masking and log filters on main paths.
- **C L0:** No path is protected: logs carry full messages and subprocess environments are unscrubbed. — [camel/agents/chat_agent.py:3701-3707](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/agents/chat_agent.py#L3701-L3707); [camel/toolkits/terminal_toolkit/terminal_toolkit.py:486](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L486) (verified)
  - *To reach the next level:* Protect at least logs and transcripts.
- **D L1:** Telemetry is opt-in, but verbose unredacted request logging is one env var away. — [camel/logger.py:30](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/logger.py#L30); [camel/utils/commons.py:589](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/utils/commons.py#L589) (verified)
  - *To reach the next level:* Logging defaults with redaction available.
- **B L0:** Long-lived provider and service keys are reachable by every subprocess and thus by the model. — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:486](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L486); [camel/interpreters/subprocess_interpreter.py:467](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/interpreters/subprocess_interpreter.py#L467) (verified)
  - *To reach the next level:* Scoped keys kept out of subprocess environments.
- **Cap:** none

### C9 Audit & traceability — 0.15 (high)

CAMEL keeps no audit trail by default. The ChatAgent returns tool-call records (name, arguments, result) to the calling code in memory, without timestamps or actor attribution, and does not write them anywhere. Model requests are logged only at INFO, below the default level. The terminal toolkit writes a command log, but inside the agent's own workspace, and it deletes existing .log files there each time the toolkit is constructed.

- **S L1:** Tool calls are captured only as in-memory ToolCallingRecord objects (no timestamps, no actor). — [camel/types/agents/tool_calling_record.py:19-35](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/types/agents/tool_calling_record.py#L19-L35) (verified)
  - *To reach the next level:* A structured persisted record of every tool call with timestamps.
- **C L1:** Records cover the ChatAgent's tool path; sub-agents and toolkits keep separate or no records. — [camel/agents/chat_agent.py:4142-4149](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/agents/chat_agent.py#L4142-L4149) (verified)
  - *To reach the next level:* Cover all built-in tools in one record.
- **D L0:** Persistent logging is opt-in; the terminal log lives inside the workspace and is wiped on construction. — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:231-234](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L231-L234); [camel/logger.py:30](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/logger.py#L30) (verified)
  - *To reach the next level:* On by default, stored outside the workspace.
- **B L0:** Nothing is durably recorded; records are lost with the process. — [camel/agents/chat_agent.py:4142-4149](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/agents/chat_agent.py#L4142-L4149) (verified)
  - *To reach the next level:* Records flushed per action with errors surfaced.
- **Cap:** G1 — Durable logging (INFO level, set_log_file, tool_log_dir) is opt-in; by default nothing is persisted.

### C10 Limits & kill switch — 0.23 (high)

The ChatAgent has an iteration cap and step/tool timeouts, but all default to unlimited (max_iteration=None, timeouts=None). The step timeout only stops waiting; the worker thread keeps running. A stop_event can halt the loop between iterations if the developer supplies one. The terminal toolkit's 20-second command timeout converts a slow command into a background process that keeps running. Workforce tasks have a 600-second default timeout.

- **S L2:** Iteration cap, step timeout and per-command timeouts are enforced in code when set; halt is a cooperative stop_event. — [camel/agents/chat_agent.py:3121-3126](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/agents/chat_agent.py#L3121-L3126); [camel/agents/chat_agent.py:2942-2956](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/agents/chat_agent.py#L2942-L2956); [camel/agents/chat_agent.py:3063](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/agents/chat_agent.py#L3063) (verified)
  - *To reach the next level:* Token/cost caps and rate limits on side-effecting tools.
- **C L1:** Limits apply per ChatAgent loop; timed-out terminal commands keep running in background and sub-agents get their own budgets. — [camel/toolkits/terminal_toolkit/terminal_toolkit.py:894-897](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/toolkits/terminal_toolkit/terminal_toolkit.py#L894-L897) (verified)
  - *To reach the next level:* Tool timeouts that actually stop the tool.
- **D L0:** max_iteration and timeouts default to None (unlimited). — [camel/agents/chat_agent.py:522](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/agents/chat_agent.py#L522); [camel/utils/constants.py:40](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/utils/constants.py#L40) (verified)
  - *To reach the next level:* Sensible finite defaults.
- **B L0:** With defaults a runaway loop has no ceiling on steps, time or spend. — [camel/agents/chat_agent.py:522](https://github.com/camel-ai/camel/blob/0106b76830c707effe48cd3da384dda38cbed92e/camel/agents/chat_agent.py#L522) (verified)
  - *To reach the next level:* Finite ceilings on steps and spend.
- **Cap:** G1 — Iteration and time limits exist but are off (None) in the default constructor.

## Rule-of-Two check
[A] untrusted input: web pages, browser content and email bodies via toolkits (camel/toolkits/gmail_toolkit.py:540) · [B] sensitive data/systems: full host environment passed to subprocesses (camel/toolkits/terminal_toolkit/terminal_toolkit.py:486); Gmail token with read/modify scopes (camel/toolkits/gmail_toolkit.py:26) · [C] state change / egress: ungated shell (camel/toolkits/terminal_toolkit/terminal_toolkit.py:1560), gmail send (camel/toolkits/gmail_toolkit.py:76), arbitrary URL fetch (camel/toolkits/web_fetch_toolkit.py:139) · Same default session? Yes

## Highest-impact improvements
1. Default TerminalToolkit require_approval to the console approver (and gate shell_write_content_to_file). — C2 D L0→L2, +0.100 before caps (Playbook 5)
2. Print the exact code/command in the interpreter confirm prompt instead of logger.info. — C2 S L1→L2, +0.075 before caps (Playbook 5)
3. Pass a minimal environment (allowlisted vars) to shell and code subprocesses instead of os.environ.copy(). — C8 B L0→L2, +0.100 before caps (Playbook 4)
4. Ship finite defaults for max_iteration and step/tool timeouts, and kill timed-out terminal processes. — C10 D L0→L2, +0.100 before caps (Playbook 3 step 3)
5. Make a container/E2B backend the default for CodeExecutionToolkit and TerminalToolkit. — C4 S L1→L2, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- CAMEL ships ~90 toolkits; only the agent loop, interpreters, terminal, code-execution, file, web-fetch, Gmail, memory, skill, MCP and Jina toolkits were read in detail. Browser toolkits, Workforce/RolePlaying internals, runtimes other than LLMGuardRuntime, and storage backends were sampled, not traced end to end.
- MCP stdio child-process environment when config env is unset relies on the MCP Python SDK's default-environment behaviour (inferred, not verified in this repo).
- No reviewer-injection text aimed at AI auditors was found in the repository.
