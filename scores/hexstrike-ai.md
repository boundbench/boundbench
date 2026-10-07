# Defense-in-Depth Score: HexStrike AI

**Repo:** https://github.com/0x4m4/hexstrike-ai · **Commit:** `d689933ff579d839c676c82b231f8e98326c5f04` · **Reviewed:** 2026-10-03
**What it is:** MCP server that lets AI agents autonomously run 150+ security tools
**Category:** Cybersecurity
**Scored configuration:** Two-process default: python3 hexstrike_server.py (Flask API) plus hexstrike_mcp.py (FastMCP client bridge) with no flags and no environment variables set.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials opt-in · persistent memory no · untrusted input yes · third party extensions yes · sub agents no · external communication yes

## Score: 0.9 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | C2-POWERBYPASS | **0.00** | High |
| C3 | Tool & action scoping | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L0 | 0.07 | C5-WORSTCASE | **0.07** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | C7-RCELOAD | **0.00** | High |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C9 | Audit & traceability | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |


The HTTP API server has no authorization, approval step, or sandbox in front of a generic shell-execution endpoint, and its default network exposure and authentication are not locked down. Any client driving the MCP bridge runs commands with the full authority of the server's OS user. Tool argument handling is not strict either, and the only bounds are a 5-minute per-command timeout and a log file. It should only run in a disposable, network-isolated VM.

## Critical gaps
- The most powerful action path, an arbitrary-command endpoint, has no approval or risk gate. (ASI02, ASI09; C2) — [hexstrike_server.py:9137](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L9137); [hexstrike_server.py:6878](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6878)
- Commands run as same-user shell children with the full environment and no isolation. (ASI05, T11; C4) — [hexstrike_server.py:6878](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6878); [hexstrike_server.py:5274](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L5274)
- A hijacked session can leak data and take irreversible actions with no human involved. (ASI01, LLM01; C5) — [hexstrike_server.py:9137](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L9137); [hexstrike_server.py:6878](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6878)
- Any caller can make the server install arbitrary remote packages without consent. (ASI04, T17; C7) — [hexstrike_server.py:14468](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L14468); [hexstrike_server.py:5726](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L5726)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

The API server holds no identity of its own: every command runs as the OS user who started it, with that user's whole environment. There is no per-request authorization anywhere in the code, and the server's default network exposure and authentication are not locked down. A hijacked agent therefore has the operator's full authority. The project's README only suggests adding authentication.

- **S L0:** Ambient authority: subprocesses run as the server's OS user with the inherited environment. — [hexstrike_server.py:6878](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6878); searched `rg -n -i 'before_request|login_required|verify_token|check_auth|require_auth|hmac\.'` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits for any request-authentication hook or token check in either file; JWT-analysis tool code does not match this pattern) (verified)
  - *To reach the next level:* Needs a dedicated, scoped identity and per-request authorization.
- **C L0:** No tool or route checks any authorization; there is no auth layer to cover paths. — searched `rg -n -i 'before_request|login_required|verify_token|check_auth|require_auth|hmac\.'` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits for any request-authentication hook or token check in either file; JWT-analysis tool code does not match this pattern); [hexstrike_server.py:9137](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L9137) (verified)
  - *To reach the next level:* Needs a check on the main tool path at minimum.
- **D L0:** The default install's network exposure and authentication are not locked down. — [README.md:650](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/README.md#L650) (verified)
  - *To reach the next level:* Needs a hardened network/authentication default and a minimal-privilege role.
- **B L0:** Failure of the (absent) authorization layer exposes arbitrary commands on the host under the operator's whole account; no independent layer remains. — [hexstrike_server.py:6878](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6878); [hexstrike_server.py:9137](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L9137) (verified)
  - *To reach the next level:* Needs write scope limited to one system with independent containment.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

The bridge exposes 151 tools and none carries a read-only or destructive hint; the most dangerous one is a documented arbitrary-command tool. The server has no confirmation step, no dry-run, and no read-only mode, so every call executes immediately and approval depends wholly on the host. The shipped sample config only has an empty always-allow list, which is a host setting rather than a server control.

- **S L0:** No risk signalling: no annotations, one generic tool mixes reads and arbitrary writes. — searched `rg -n -i 'readOnlyHint|destructiveHint|dry_run|dry-run'` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits for risk annotations or dry-run support); [hexstrike_mcp.py:3969](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_mcp.py#L3969) (verified)
  - *To reach the next level:* Needs accurate read/write annotations and separate tools.
- **C L0:** The most powerful path, arbitrary shell execution, has no gate at all. — searched `rg -n -i 'approv|confirm_|require_confirm'` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits for approval or confirmation logic in either file; 'escalate_to_human' recovery actions are error-handling labels and do not match); [hexstrike_server.py:9137](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L9137) (verified)
  - *To reach the next level:* Needs every tool path to cross a gate.
- **D L0:** No approval mechanism exists to be on by default. — searched `rg -n -i 'approv|confirm_|require_confirm'` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits for approval or confirmation logic in either file; 'escalate_to_human' recovery actions are error-handling labels and do not match) (verified)
  - *To reach the next level:* Needs a default-on server-side confirmation.
- **B L0:** Wrongly approved or ungated calls can run irreversible, high-impact commands with no undo, preview, or rate limit. — [hexstrike_server.py:6878](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6878); [hexstrike_server.py:8985](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L8985) (verified)
  - *To reach the next level:* Needs checkpoints, previews, and rate limits.
- **Cap:** C2-POWERBYPASS — The generic command endpoint, the most powerful action path, has no gate in the default configuration.

### C3 Tool & action scoping — 0.00 (high)

Tool argument handling is not strict, and there are no schema constraints or scope allowlist. Checks are limited to 'field is present'. The generic command endpoint, a Python-script execution endpoint, a package-install endpoint, and a file API are all on by default with no way to select a narrower tool set.

- **S L0:** Raw passthrough: the generic endpoint takes an arbitrary shell string, and argument handling in other tools is not strict. (verified)
  - *To reach the next level:* Needs at least denylist or regex filtering; allowlists for L3.
- **C L0:** No tool validates inputs beyond required-field presence; file API path handling is not a strict boundary. (verified)
  - *To reach the next level:* Needs validation in a few tools at least.
- **D L0:** All tool groups, shell, script execution, package install, and file write/delete are enabled with no selection mechanism. — [hexstrike_server.py:14498](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L14498); [hexstrike_server.py:14468](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L14468); [hexstrike_server.py:9137](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L9137) (verified)
  - *To reach the next level:* Needs individually disableable dangerous tools.
- **B L0:** A misused tool reaches any command and any host the machine can reach, with no bounds. — [hexstrike_server.py:6878](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6878); [hexstrike_server.py:9137](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L9137) (verified)
  - *To reach the next level:* Needs scoping to a project or workspace.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

Every command runs as a same-user child of the API process through a shell, with the full inherited environment and no container, namespace, or user separation. The only boundary is the host itself. The optional headless browser tool is launched with its own sandbox and web-security protections switched off.

- **S L0:** No isolation primitive: same-user subprocess with shell=True. — [hexstrike_server.py:6878](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6878); searched `rg -n -i 'docker run|--privileged|seccomp|unshare|nsjail|firejail|bwrap|chroot'` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits for any container, namespace, or sandbox primitive) (verified)
  - *To reach the next level:* Needs at least filtering or a separate working directory.
- **C L0:** No execution path is sandboxed, including generic command, Python script, and package install paths. — [hexstrike_server.py:5274](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L5274); [hexstrike_server.py:5726](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L5726); searched `rg -n -i 'docker run|--privileged|seccomp|unshare|nsjail|firejail|bwrap|chroot'` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits for any container, namespace, or sandbox primitive) (verified)
  - *To reach the next level:* Needs the main exec tool sandboxed.
- **D L0:** There is no sandbox to be on by default. — searched `rg -n -i 'docker run|--privileged|seccomp|unshare|nsjail|firejail|bwrap|chroot'` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits for any container, namespace, or sandbox primitive) (verified)
  - *To reach the next level:* Needs a sandbox enabled by default.
- **B L0:** Execution is host-equivalent: full environment and home directory reachable, full network, no resource limits. — searched `rg -n -i 'env='` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits: no subprocess call scrubs or sets the environment, so children inherit everything); [hexstrike_server.py:6878](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6878) (verified)
  - *To reach the next level:* Needs a workspace-only mount, scrubbed environment, and limits.
- **Cap:** none
- **Notes:** The headless browser helper sets Chrome flags that disable its sandbox and web security (hexstrike_server.py:13640, 13651).

### C5 Untrusted input blast radius — 0.07 (high)

Tool output, including content fetched from scan targets and third-party responses, returns to the model as plain stdout/stderr strings with no provenance flag or untrusted marker. The same session can both read that content and run any command, reach the network, and delete files, with no human step. A hijacked session can therefore leak data and take irreversible actions unattended.

- **S L1:** Outputs are plain stdout/stderr dictionaries with no provenance or untrusted flag; no directives to the model were found in sampled outputs. — [hexstrike_server.py:6804](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6804); searched `rg -n -i 'untrusted|provenance|taint'` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits for untrusted-content marking) (verified)
  - *To reach the next level:* Needs structured content separated from metadata and a provenance marker.
- **C L0:** No source is distinguished from the principal's instructions; scan-target output enters context like any other result. — [hexstrike_server.py:6804](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6804) (verified)
  - *To reach the next level:* Needs at least one source handled.
- **D L0:** No limit exists to be on by default. — searched `rg -n -i 'untrusted|provenance|taint'` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits) (verified)
  - *To reach the next level:* Needs a default-on limit.
- **B L0:** Default config lets a hijacked session exfiltrate data over the network and run irreversible commands with no human involved. — [hexstrike_server.py:6878](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6878); [hexstrike_server.py:9137](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L9137) (verified)
  - *To reach the next level:* Needs approval on egress and irreversible actions.
- **Cap:** C5-WORSTCASE — B is L0: unattended leak plus irreversible actions in the default configuration.
- **Notes:** S at L1 reflects the baseline tool-server anchor (plain outputs, no directives found); D is L0 because no limiting control exists at all, so G1 (off-by-default control) does not apply.

### C6 Memory, context & configuration integrity — 0.15 (high)

No agent memory or auto-loaded instruction files exist. The server does keep global state that persists outside any session: an in-memory command-result cache keyed by command string, a shared file store under a temp directory, and persistent Python virtual environments. These stores are not partitioned per caller, and any client can write, delete, and execute their contents. Nothing from them is automatically re-injected into model context, and no workspace file is loaded as configuration.

- **S L1:** Writes to the file store and cache are logged but not validated; nothing loads silently into context. — [hexstrike_server.py:8936](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L8936); [hexstrike_server.py:8952](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L8952) (verified)
  - *To reach the next level:* Needs provenance or gating on writes.
- **C L1:** No store is controlled: cache, file store and venvs are all open to any caller. — [hexstrike_server.py:8931](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L8931); [hexstrike_server.py:5708](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L5708) (verified)
  - *To reach the next level:* Needs at least the main store controlled.
- **D L0:** All state is one global namespace shared by every caller; there are no users or tenants. — [hexstrike_server.py:8931](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L8931); searched `rg -n -i 'load_dotenv|dotenv'` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits: no .env or workspace configuration is loaded) (verified)
  - *To reach the next level:* Needs per-caller namespaces enforced in code.
- **B L0:** Persisted files and environments are shared across sessions and callers and can be executed through the command and script endpoints. — [hexstrike_server.py:14520](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L14520) (verified)
  - *To reach the next level:* Needs session-scoped or reviewable state.
- **Cap:** none
- **Notes:** Not scored as structural absence because persistence the caller can influence does exist. No memory is read back into model context, which is why the levels stay low rather than L0 on S and C. D is L0 because the stores are globally shared, not because an opt-in control exists, so G1 does not apply.

### C7 Third-party extensions — 0.00 (high)

The server exposes an endpoint that installs any named Python package into a persistent virtual environment on request, and a companion endpoint that runs caller-supplied Python scripts in it. Installation is unpinned, unverified, and needs no consent, and package install steps run code with the server user's authority and full environment. The roughly 150 external security binaries are installed separately by the operator and are not fetched by the server.

- **S L0:** Model- or caller-chosen package names are installed without verification or pinning. — [hexstrike_server.py:5726](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L5726); [hexstrike_server.py:14468](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L14468) (verified)
  - *To reach the next level:* Needs user-chosen but at least pinned sources.
- **C L0:** No extension type is verified, including packages installed at runtime. — [hexstrike_server.py:5726](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L5726) (verified)
  - *To reach the next level:* Needs one type verified.
- **D L0:** Installation is enabled by default and reachable by any caller with no consent step. — [hexstrike_server.py:14468](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L14468); searched `rg -n -i 'approv|confirm_|require_confirm'` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits for any consent logic) (verified)
  - *To reach the next level:* Needs an explicit consent flow showing what will run.
- **B L0:** Installed code runs as the same user with the full environment, in-process with the child pip run. — [hexstrike_server.py:5726](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L5726); searched `rg -n -i 'env='` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits: no subprocess call scrubs or sets the environment) (verified)
  - *To reach the next level:* Needs a separate process with a scrubbed environment.
- **Cap:** C7-RCELOAD — By default any caller can make the server fetch and install arbitrary remote packages, whose install scripts execute, without consent.

### C8 Secrets & sensitive-data protection — 0.00 (high)

Tools take credentials such as passwords and API keys as ordinary arguments and splice them into command lines. Every command line and all its output are written at info level to the console and to a log file with no redaction. Every subprocess inherits the server's full environment. The debug mode of the client also logs complete request bodies. No secret handling or masking code exists, and the log file is created in the working directory.

- **S L0:** Credential-bearing command lines and outputs are logged verbatim with no masking. — [hexstrike_server.py:6872](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6872); [hexstrike_server.py:79](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L79); searched `rg -n -i 'redact|sanitize_log|mask_secret'` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits for redaction helpers; the matches for 'mask' in other patterns are hashcat attack-mode parameters) (verified)
  - *To reach the next level:* Needs at least env-var sourcing with some masking.
- **C L0:** No path is protected: logs, outputs, errors, and subprocess environments all carry secrets as-is. — [hexstrike_server.py:6806](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6806); [hexstrike_mcp.py:234](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_mcp.py#L234) (verified)
  - *To reach the next level:* Needs one path protected.
- **D L0:** Verbose payload logging of commands and output is on by default with no off switch for redaction. — [hexstrike_server.py:6872](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6872) (verified)
  - *To reach the next level:* Needs redaction on by default.
- **B L0:** Long-lived operator keys in the inherited environment are reachable by every subprocess. — searched `rg -n -i 'env='` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits: no subprocess call scrubs or sets the environment); [hexstrike_server.py:6878](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6878) (verified)
  - *To reach the next level:* Needs scoped, short-lived credentials.
- **Cap:** none

### C9 Audit & traceability — 0.20 (high)

Commands, their output, and timings are written as free-text lines to the console and to hexstrike.log, which gives a basic trail for the main execution path. Records are unstructured, carry no caller identity, approver, or correlation ID, and some direct subprocess calls in analysis helpers do not log their argument vectors. The log file sits in the working directory, where the same command endpoint can alter or delete it, and if the file cannot be opened the server silently continues with console-only logging.

- **S L1:** Unstructured log lines record commands and results, with no structured fields. — [hexstrike_server.py:6872](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6872); [hexstrike_server.py:79](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L79) (verified)
  - *To reach the next level:* Needs a structured per-call record with arguments, status, timestamps.
- **C L1:** The main shell execution path logs; direct subprocess.run helpers do not log their own argument vectors, and nothing records approvals or callers. — [hexstrike_server.py:16398](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L16398); [hexstrike_server.py:6872](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6872) (verified)
  - *To reach the next level:* Needs all built-in tools recorded.
- **D L1:** On by default but the file lands in the working directory, which the command and file endpoints can edit or delete. — [hexstrike_server.py:79](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L79) (verified)
  - *To reach the next level:* Needs a location outside the agent's reach.
- **B L0:** On PermissionError the server falls back to console-only logging and actions proceed with no durable record. — [hexstrike_server.py:82](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L82); [hexstrike_server.py:84](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L84) (verified)
  - *To reach the next level:* Needs errors surfaced and per-action flush.
- **Cap:** none

### C10 Limits & kill switch — 0.45 (high)

Each command run through the main executor has a fixed 5-minute timeout, the file API caps file size, and the async pool is capped at 32 workers. There is no rate limit, no cap on concurrent requests, no output-size cap, and no overall budget. The timeout terminates only the shell process because commands are not started in their own process group, so child processes can outlive both the timeout and the terminate endpoint. A runaway caller can issue unlimited consecutive commands.

- **S L2:** Server-enforced per-command timeout and a file-size cap exist; output size, rate and concurrency are unbounded. — [hexstrike_server.py:6664](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6664); [hexstrike_server.py:6906](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6906); [hexstrike_server.py:8934](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L8934); searched `rg -n -i 'RateLimiter|flask_limiter|Limiter\(|@limiter'` in `hexstrike_server.py hexstrike_mcp.py` → 0 hits (zero hits for request rate limiting) (verified)
  - *To reach the next level:* Needs caps on every operation plus concurrency or rate limits.
- **C L2:** The main executor and tool endpoints share the 300s timeout; the async pool path waits with no timeout and spawned children are not covered. — [hexstrike_server.py:5292](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L5292); [hexstrike_server.py:6934](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6934) (verified)
  - *To reach the next level:* Needs background tasks and spawned processes inside the same budget.
- **D L2:** Timeout is a fixed module constant callers cannot change per request, but a caller can background work from the shell string to outlive it. — [hexstrike_server.py:6664](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6664) (verified)
  - *To reach the next level:* Needs limits that delegation or shell backgrounding cannot escape.
- **B L1:** Stopping signals only the shell pid with no process-group kill in the main executor, leaving children running, and there is no ceiling on the number or total volume of commands. — [hexstrike_server.py:6876](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L6876); [hexstrike_server.py:5625](https://github.com/0x4m4/hexstrike-ai/blob/d689933ff579d839c676c82b231f8e98326c5f04/hexstrike_server.py#L5625) (verified)
  - *To reach the next level:* Needs tight ceilings and a halt that cancels in-flight work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Scan targets and their responses flow back as tool output (hexstrike_server.py:6804) · [B] sensitive data/systems: Server inherits the operator's full environment and filesystem (hexstrike_server.py:6876) · [C] state change / egress: Generic command endpoint with shell=True (hexstrike_server.py:9137, 6878) · Same default session? Yes

## Highest-impact improvements
1. Harden the default network exposure and require an authenticated caller before any tool or command endpoint runs. — C1 D L0→L2, +0.100 before caps
2. Run every command in a hardened, ephemeral container with no host mounts, a scrubbed environment, and egress limited to an operator-set scope. — C4 S L0→L3, +0.225 before caps
3. Validate tool targets against an operator-defined scope allowlist; remove the generic command endpoint from the default tool set. — C3 S L0→L3, +0.225 before caps
4. Add risk annotations plus a server-enforced confirmation or read-only mode that the host must complete before mutating tools run. — C2 S L0→L3, +0.225 before caps
5. Write a structured per-call audit record (caller, argv, status) to a location the tools cannot edit, and redact credential-bearing arguments. — C9 S L1→L3, +0.150 before caps

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was installed, built, executed, or probed.
- Review covered the two top-level Python files (hexstrike_server.py, 17289 lines, and hexstrike_mcp.py, 5470 lines); the roughly 150 per-tool endpoints were sampled for shared patterns, not read one by one.
- The MCP bridge declares 151 tool registrations; the C5 tool-description review was a sampled search, not a line-by-line read of every docstring.
- Scored as a tool server: the host application owns any approval prompt, and the sample MCP config's empty alwaysAllow list was not credited as a server control.
- No reviewer-directed instructions were found in the repository text.
