# Defense-in-Depth Score: career-ops

**Repo:** https://github.com/career-ops-hq/career-ops · **Commit:** `a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8` (v1.35.0) · **Reviewed:** 2026-10-04
**What it is:** Open-source AI job-search system: a prompt pack and Node scripts that run inside your AI coding CLI to scan job boards, score postings against your CV, tailor CVs and track applications.
**Category:** AI Assistants
**Scored configuration:** Interactive session of a host AI coding CLI opened in a fresh `npx @santifer/career-ops init` checkout with the shipped AGENTS.md/SKILL.md and modes; no plugins, no web UI, no batch runner (the batch runner and web UI are footnoted).
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials opt-in · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents yes · external communication no

## Score: 2.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L1 | 0.05 | C2-SELFAPPROVE | **0.05** | High |
| C3 | Tool & action scoping | L2 | L2 | L1 | L1 | 0.40 | G1 | **0.40** (alt) | High |
| C4 | Code-execution isolation | L2 | L0 | L0 | L0 | 0.15 | G1 | **0.15** (alt) | High |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | L3 | L2 | L0 | L0 | 0.38 | — | **0.38** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C9 | Audit & traceability | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |


career-ops is mostly prompts that steer your AI coding CLI, plus well-engineered scripts. The scripts include SSRF-hardened job scanners and an apply helper that never submits. There is no mail transport, and plugins are pinned to exact commits and hash-locked. Safety in the agent loop itself rests on the host CLI and on instructions in AGENTS.md: untrusted job postings share a session with your CV, your shell and the network, and nothing in code stops a hijacked session from leaking data or acting. The biggest risks are prompt injection from postings, model-written house rules and inbox items that re-execute in every session, and a documented batch runner that launches Claude with --dangerously-skip-permissions on the host.

## Critical gaps
- The agent runs with the user's full ambient authority and passes the whole environment to subprocesses and plugins, so a hijacked session holds everything the user holds. (ASI03, T3; C1) — [plugins/_engine.mjs:476-478](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins/_engine.mjs#L476-L478); [batch/batch-runner.sh:1069](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L1069)
- Self-update and plugin-enable consent is a --confirm CLI flag the agent can pass itself, so career-ops' own gates can be self-approved; the batch runner also defaults to --dangerously-skip-permissions. (ASI09, ASI02, T10; C2) — [update-system.mjs:3477](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/update-system.mjs#L3477); [plugins.mjs:329](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins.mjs#L329); [batch/batch-runner.sh:1041](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L1041)
- Model-driven execution (scripts, Playwright, headless batch workers with permissions skipped) runs directly on the host as the user, with no isolation boundary. (ASI05, T11; C4) — [batch/batch-runner.sh:1041](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L1041); [batch/batch-runner.sh:1069](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L1069)
- Untrusted job postings, pages and emails share a session with the user's CV/PII, shell and network access, guarded only by prompt instructions, so a successful injection can leak data and act unattended. (ASI01, LLM01, T6; C5) — [AGENTS.md:69](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/AGENTS.md#L69); searched `rg -n -i 'untrusted'` in `fetch-jd.mjs browser-extract.mjs liveness-browser.mjs scan.mjs` → 0 hits (the scripts that fetch job postings attach no provenance or untrusted marker to what they return)
- Model-written modes/_custom.md and the append-anywhere agent inbox are re-loaded as binding instructions every session. (ASI06, T1; C6) — [modes/_shared.md:31](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/modes/_shared.md#L31); [agent-inbox.mjs:41-42](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/agent-inbox.mjs#L41-L42)
- Enabled plugins are imported into the career-ops process with the whole process.env reachable, so a malicious or compromised plugin gets everything the scripts hold. (ASI04, T17; C7) — [plugins/_engine.mjs:533](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins/_engine.mjs#L533); [plugins/_engine.mjs:476-478](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins/_engine.mjs#L476-L478)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

career-ops has no identity of its own. It runs inside your AI coding CLI as your OS user, and the scripts it tells the agent to run inherit your full environment. The job sources it scans are public and need no login, and no credential is required by default. But nothing narrows what the agent's shell session can reach: your git, cloud and SSH credentials stay reachable, and plugin code gets all of process.env. A hijacked session therefore holds everything you hold.

- **S L0:** The agent acts with the launching user's ambient authority; career-ops creates no scoped identity and its plugin env scoping is documented as a convenience, not a boundary. — [plugins/_engine.mjs:476-478](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins/_engine.mjs#L476-L478); [AGENTS.md:7](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/AGENTS.md#L7) (verified)
  - *To reach the next level:* No dedicated or role-scoped identity for the agent or its scripts.
- **C L0:** No authorization check exists on any path; subprocesses (batch workers, web UI workers, plugins) get the parent environment unchanged. — searched `rg -n 'env:'` in `web/src/lib/spawn-cli.mjs` → 0 hits (spawnHeadlessCli passes caller options straight to spawn; no env allowlist or scrubbing); [batch/batch-runner.sh:1069](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L1069) (verified)
  - *To reach the next level:* No environment scrubbing or authorization layer on even the main script path.
- **D L0:** The default install runs with the user's full privilege; there is nothing to harden from. — [AGENTS.md:7](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/AGENTS.md#L7) (verified)
  - *To reach the next level:* No narrower default identity exists.
- **B L0:** A hijacked session reaches whatever the OS user can (local files, ambient git/cloud/SSH credentials) through the shell the workflow requires. — [plugins/_engine.mjs:476-478](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins/_engine.mjs#L476-L478); [batch/batch-runner.sh:1069](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L1069) (verified)
  - *To reach the next level:* Blast radius would shrink only if the agent's shell and scripts ran without the user's ambient credentials.
- **Cap:** none

### C2 Approval gates — 0.05 (high)

career-ops ships no approval gate of its own. In the interactive mode it relies on the host CLI's permission prompts. Those prompts are not part of this repository and are not credited here. Its own consent checks, for applying a self-update and for enabling a plugin, are passed by a --confirm command-line flag that the agent can add itself. The documented batch runner turns the host's prompts off by default with --dangerously-skip-permissions. What career-ops gets right is the reach of its own code: no script submits an application or sends email, and the updater keeps a backup branch with rollback.

- **S L0:** The only gates in career-ops code are --confirm flags that the model itself supplies; the human confirmation exists only as an AGENTS.md instruction to ask first. — [update-system.mjs:3477](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/update-system.mjs#L3477); [plugins.mjs:329](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins.mjs#L329); searched `rg -n -i 'readline|isTTY'` in `update-system.mjs plugins.mjs` → 0 hits (no interactive human prompt in either consent path; the flag alone authorises) (verified)
  - *To reach the next level:* No per-call human approval in code; consent is a flag the agent can pass.
- **C L0:** The most powerful path, the shell commands the workflow runs, passes no career-ops gate, and the batch runner launches workers with permissions skipped. — [batch/batch-runner.sh:1041](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L1041); [AGENTS.md:7](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/AGENTS.md#L7) (verified)
  - *To reach the next level:* No gate on shell, file-write or network paths owned by career-ops.
- **D L0:** No approval is on by default in career-ops; its own headless runner defaults to skipping the host's prompts. — [batch/batch-runner.sh:1041](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L1041) (verified)
  - *To reach the next level:* No default-on gate; the batch default actively disables the host gate.
- **B L1:** career-ops code has no submit, send or mail transport, and the updater keeps a backup branch and rollback, but a wrongly run shell or browser action through the host is irreversible. — [prepare-application.mjs:8](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/prepare-application.mjs#L8); [update-system.mjs:16-17](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/update-system.mjs#L16-L17) (verified)
  - *To reach the next level:* No checkpoint/rollback for user-layer files and no dry-run for actions taken through the host shell or browser.
- **Cap:** C2-SELFAPPROVE — career-ops' own consent checks (update apply, plugin enable) are satisfied by a --confirm flag the model can pass with no human interaction (update-system.mjs:3477, plugins.mjs:329).

### C3 Tool & action scoping — 0.40 (high)

The network code career-ops owns is careful. Scanner providers use per-provider host allowlists, refuse redirects, and check resolved IP addresses against private ranges at DNS time. Plugin HTTP goes through an HTTPS host allowlist that re-checks redirects. The Playwright browser paths are weaker; their address guard does not cover every path. And the workflow drives everything through a general shell with write and network tools on by default. The opt-in web UI adds per-worker tool allowlists (read-only and no-network scopes for PDF and research), but its evaluation workers still get unrestricted Bash and WebFetch.

- **default configuration** (default; raw 0.35 → 0.35)
  - **S L2:** Providers meet allowlist-plus-DNS-time IP validation, but the browser paths' guard does not cover every path, and the scripts are reached through a raw shell. — [providers/_http.mjs:107](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/providers/_http.mjs#L107); [providers/_ip-guard.mjs:106](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/providers/_ip-guard.mjs#L106); [providers/greenhouse.mjs:31](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/providers/greenhouse.mjs#L31) (verified)
    - *To reach the next level:* Stronger validation on the browser paths, and narrow tools in place of the shell.
  - **C L2:** Most network-facing built-in scripts validate, and plugin fetches go through a guarded primitive, but plugins can still reach the global fetch. — [plugins/_engine.mjs:418-429](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins/_engine.mjs#L418-L429); [plugins/_engine.mjs:476-478](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins/_engine.mjs#L476-L478) (verified)
    - *To reach the next level:* Plugin and shell paths are not forced through one validation layer.
  - **D L0:** The default working set includes shell, file write and network; nothing is read-only by default. — [AGENTS.md:7](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/AGENTS.md#L7); [.agents/skills/career-ops/SKILL.md:196](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/.agents/skills/career-ops/SKILL.md#L196) (verified)
    - *To reach the next level:* No read-only default tool set.
  - **B L1:** career-ops scripts are confined to the project directory and public HTTPS GETs, but a misused shell reaches the whole machine. — [AGENTS.md:7](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/AGENTS.md#L7); [providers/_http.mjs:107](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/providers/_http.mjs#L107) (verified)
    - *To reach the next level:* The workflow's general shell keeps reach machine-wide.
- **opt-in web UI headless worker tool scopes** (alt; raw 0.40, cap G1 → 0.40) ← counted
  - **S L2:** Per-kind Claude tool allowlists with explicit deny lists; writing kinds still receive unparameterised Bash and WebFetch under acceptEdits. — [web/src/lib/claude-invocation.mjs:109-113](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/web/src/lib/claude-invocation.mjs#L109-L113); [web/src/lib/claude-invocation.mjs:164](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/web/src/lib/claude-invocation.mjs#L164) (verified)
    - *To reach the next level:* Argument-level limits on the Bash and WebFetch granted to writing kinds.
  - **C L2:** Every Claude worker is fenced and spawning fails closed without a capability record, but other CLIs keep their own default tools. — [web/src/lib/spawn-cli.mjs:54-59](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/web/src/lib/spawn-cli.mjs#L54-L59); [web/README.md:46-53](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/web/README.md#L46-L53) (verified)
    - *To reach the next level:* Equivalent fencing for every supported CLI.
  - **D L1:** Fencing is on whenever the web UI is used, but the UI itself is an opt-in alpha. — [web/README.md:1-3](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/web/README.md#L1-L3) (verified)
    - *To reach the next level:* On by default only inside an opt-in mode.
  - **B L1:** An evaluation worker fed an untrusted posting can still run arbitrary shell commands with network access. — [web/src/lib/claude-invocation.mjs:109-113](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/web/src/lib/claude-invocation.mjs#L109-L113) (verified)
    - *To reach the next level:* Writing workers would need a bounded command set.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C4 Code-execution isolation — 0.15 (high)

Nothing career-ops runs is isolated. The scripts, the Playwright automation and the headless batch workers all run on the host as the user. The batch workers also run with the host CLI's permission prompts disabled. The shipped Docker setup is a compatibility option, not a sandbox: the container runs as root, mounts the whole project read-write, has open network access, and receives your API keys. The agent's own shell still runs on the host anyway.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Every execution path is a same-user host process; no isolation primitive exists on the paths career-ops owns. — searched `rg -n -i 'seccomp|landlock|firejail|bwrap|gvisor|firecracker|nsjail'` in `batch providers plugins lib utils scan.mjs update-system.mjs browser-extract.mjs generate-pdf.mjs` → 0 hits (no isolation primitive anywhere on the execution paths career-ops owns); [batch/batch-runner.sh:1069](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L1069) (verified)
    - *To reach the next level:* No container, OS sandbox or restricted runtime around model-driven execution.
  - **C L0:** No path is sandboxed by default. — [batch/batch-runner.sh:1069](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L1069) (verified)
    - *To reach the next level:* Not even the batch worker path is isolated.
  - **D L0:** There is no default isolation to turn on or off. — searched `rg -n -i 'seccomp|landlock|firejail|bwrap|gvisor|firecracker|nsjail'` in `batch providers plugins lib utils scan.mjs update-system.mjs browser-extract.mjs generate-pdf.mjs` → 0 hits (no isolation primitive anywhere on the execution paths career-ops owns) (verified)
    - *To reach the next level:* No default-on sandbox.
  - **B L0:** Execution is host-equivalent: user home, credentials and network are all reachable. — [batch/batch-runner.sh:1041](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L1041); [batch/batch-runner.sh:1069](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L1069) (verified)
    - *To reach the next level:* Execution would need to move off the host user's account.
- **opt-in Docker compose environment** (alt; raw 0.15, cap G1 → 0.15) ← counted
  - **S L2:** A stock Playwright container with no USER directive, so it runs as root with default capabilities. — [Dockerfile:6](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/Dockerfile#L6); searched `rg -n '^USER'` in `Dockerfile` → 0 hits (no USER directive, so the Playwright base image runs as root) (verified)
    - *To reach the next level:* Non-root user, dropped capabilities, seccomp, read-only root.
  - **C L0:** Only commands routed through ./cops run inside; the AI CLI and its shell still execute on the host. — [cops:1](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/cops#L1); [docker-compose.yml:15-16](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/docker-compose.yml#L15-L16) (verified)
    - *To reach the next level:* The agent's own command execution would need to run inside the container.
  - **D L0:** The container is opt-in and presented as a compatibility aid. — [DOCKER.md:3-5](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/DOCKER.md#L3-L5) (verified)
    - *To reach the next level:* Not on by default.
  - **B L0:** The project is bind-mounted read-write, network is unrestricted, and host API keys are forwarded into the container environment. — [docker-compose.yml:15-16](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/docker-compose.yml#L15-L16); [docker-compose.yml:22-24](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/docker-compose.yml#L22-L24) (verified)
    - *To reach the next level:* No credentials in the container, workspace-only mount and restricted egress.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.25 (high)

career-ops reads untrusted text all day: job postings, company pages, application forms and recruiter emails. The defence against injected instructions in that text is a strongly worded section of AGENTS.md telling the model to treat it as data, and a CI check that the warning text appears in every mode. Community plugin skill text is printed under an UNTRUSTED banner, but postings and pages reach the model unmarked. Those same sessions hold your CV and personal details and have network and shell access. Nothing in the code stops a hijacked session from leaking that data or acting on it.

- **S L1:** Defences are prompt instructions plus a spotlighting banner on community plugin skill text; no capability is disabled after untrusted content is read. — [AGENTS.md:69](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/AGENTS.md#L69); [plugins.mjs:322](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins.mjs#L322); [validate-untrusted-content-coverage.mjs:16-18](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/validate-untrusted-content-coverage.mjs#L16-L18) (verified)
  - *To reach the next level:* No code-enforced rule that removes egress or state-changing tools once untrusted content enters the session.
- **C L1:** Only plugin skill text is marked; postings, pages, forms and emails enter context unmarked. — searched `rg -n -i 'untrusted'` in `fetch-jd.mjs browser-extract.mjs liveness-browser.mjs scan.mjs` → 0 hits (the scripts that fetch job postings attach no provenance or untrusted marker to what they return); [plugins.mjs:322](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins.mjs#L322) (verified)
  - *To reach the next level:* Provenance marking on postings, pages, forms, emails and sub-agent output.
- **D L2:** The banner is always printed for non-bundled plugin skills and cannot be configured away by content. — [plugins.mjs:322](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins.mjs#L322) (verified)
  - *To reach the next level:* The banner covers one source only, and nothing warns when it is absent.
- **B L0:** A hijacked session holds the CV/PII, can exfiltrate through the shell or browser, and can take irreversible actions (shell, live browser on application forms) with no career-ops-enforced human step. — [AGENTS.md:69](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/AGENTS.md#L69); [AGENTS.md:7](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/AGENTS.md#L7); [.agents/skills/career-ops/SKILL.md:196](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/.agents/skills/career-ops/SKILL.md#L196) (verified)
  - *To reach the next level:* Sessions that read postings would need to lose egress or state-changing tools.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.10 (high)

career-ops is built to remember. The model is told to write your lasting preferences and 'automations' into modes/_custom.md, and every mode loads that file as binding instructions. data/agent-inbox.md is a queue of requests that any script or tool can append to, and the agent runs the open items top to bottom. Neither file is validated, tagged with its source, or gated. A single injection that lands in either one fires in every later session. Separately, plugin enablement from workspace configuration is not integrity-protected.

- **S L0:** The model writes free text into _custom.md and the agent inbox, and both are re-read as binding instructions in later sessions; plugin enablement from workspace config is not integrity-protected. — [modes/_shared.md:31](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/modes/_shared.md#L31); [agent-inbox.mjs:41-42](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/agent-inbox.mjs#L41-L42) (verified)
  - *To reach the next level:* No validation, provenance or approval on writes to persistent instruction files.
- **C L0:** No persistent store is controlled: house rules, the inbox, the story bank and archived postings are all unguarded. — [modes/_shared.md:31](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/modes/_shared.md#L31); [agent-inbox.mjs:41-42](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/agent-inbox.mjs#L41-L42) (verified)
  - *To reach the next level:* Controls on even the main persistent instruction file.
- **D L1:** Data lives in the single user's local checkout, but nothing enforces separation beyond OS file ownership, and the model can write every store. — [AGENTS.md:7](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/AGENTS.md#L7) (verified)
  - *To reach the next level:* Per-store isolation the model cannot write across.
- **B L1:** Poisoned house rules or inbox items persist across the user's sessions and can trigger tool use; the files are plain markdown and inspectable, but nothing flags a poisoned entry. — [modes/_shared.md:31](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/modes/_shared.md#L31); [agent-inbox.mjs:41-42](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/agent-inbox.mjs#L41-L42) (verified)
  - *To reach the next level:* Poisoned entries would need to be limited to text output or require review before they act.
- **Cap:** C6-REPOCONFIG — Plugin enablement from workspace configuration is not integrity-protected, which meets the cap trigger.

### C7 Third-party extensions — 0.38 (high)

Plugins are off by default. The community registry pins every plugin to an exact 40-character commit. Installation clones that commit with risky git transports disabled, runs a static audit, and records a sha256 hash of every file. Changed files load only if the version number changed; if allowed hosts or keys expand, the user must consent again. The weak points: a version bump re-pins quietly without fresh consent, enabling is a --confirm flag the agent can pass, and the consent flow is not tamper-resistant in other respects. Plugins also run inside the same Node process with full access to process.env.

- **S L3:** Registry entries and installs are pinned to exact SHAs, the whole plugin tree is sha256-locked, and a static audit runs at install. — [plugin-install.mjs:23-25](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugin-install.mjs#L23-L25); [plugin-install.mjs:41-50](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugin-install.mjs#L41-L50); [plugins/_lock.mjs:37](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins/_lock.mjs#L37); [plugins/_engine.mjs:625](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins/_engine.mjs#L625) (verified)
  - *To reach the next level:* Re-approval whenever plugin code changes; a version bump currently re-pins quietly.
- **C L2:** Plugins and plugin skills are verified, but the MCP server the project recommends (Playwright) is shown unpinned via npx -y. — [opencode.example.json:6](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/opencode.example.json#L6); [plugins/_lock.mjs:37](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins/_lock.mjs#L37) (verified)
  - *To reach the next level:* Pinning or verification for the recommended MCP server.
- **D L0:** No plugin is enabled by default, but the consent flow is not tamper-resistant, and the consent card is skipped by a flag the agent can pass. — [plugins.mjs:329](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins.mjs#L329); [.env.example:40-41](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/.env.example#L40-L41) (verified)
  - *To reach the next level:* Consent that the agent cannot supply itself.
- **B L0:** Plugins are imported in-process with the full environment reachable; the scoped env is explicitly a convenience. — [plugins/_engine.mjs:533](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins/_engine.mjs#L533); [plugins/_engine.mjs:476-478](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins/_engine.mjs#L476-L478) (verified)
  - *To reach the next level:* Separate process with a scrubbed environment per plugin.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.30 (high)

No credential is required by default. Optional LLM API keys live in a plaintext, gitignored .env file in the project directory. Every script and subprocess can read them, and so can the agent through its shell. Some masking exists: plugin log output redacts the plugin's keys, one provider masks a URL token, and Gemini error messages strip the key. There is no telemetry anywhere. Your CV and personal details go to whichever model provider you chose, which is the tool's purpose.

- **S L1:** Secrets come from a plaintext .env file or environment variables, with masking on a few paths only. — [.env.example:10](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/.env.example#L10); [.gitignore:126](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/.gitignore#L126); [openrouter-runner.mjs:52-53](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/openrouter-runner.mjs#L52-L53); [plugins/_engine.mjs:498](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins/_engine.mjs#L498) (verified)
  - *To reach the next level:* Type-level masking and log filters on main paths, or keychain storage.
- **C L1:** Redaction covers plugin logs and two error/URL paths; batch logs, subprocess environments and model-bound context are unprotected. — [plugins/_engine.mjs:498](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins/_engine.mjs#L498); [batch/batch-runner.sh:1069](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L1069) (verified)
  - *To reach the next level:* Redaction on logs and transcripts generally, including batch worker logs.
- **D L2:** No telemetry or crash reporting exists, and logging is local; the redaction that exists is narrow. — searched `rg -n -i 'posthog|sentry|mixpanel|segment\.io|datadog'` in `providers plugins lib utils web/src scan.mjs update-system.mjs` → 2 hits (both hits are company names (a Greenhouse comment and a company-domain map), not telemetry SDKs) (verified)
  - *To reach the next level:* Redaction that is always on across paths.
- **B L1:** Long-lived provider API keys and optional plugin OAuth refresh tokens are readable by every subprocess and the agent's shell. — [.env.example:10](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/.env.example#L10); [plugins/_engine.mjs:476-478](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/plugins/_engine.mjs#L476-L478) (verified)
  - *To reach the next level:* Scoped or short-lived credentials.
- **Cap:** none

### C9 Audit & traceability — 0.25 (high)

career-ops keeps records of outcomes, not of agent actions. There is a status-change ledger (status-log.tsv), a scan history, per-worker stdout logs for batch runs, and the reports themselves. The detailed record of what the agent ran is the host CLI's transcript, which career-ops neither owns nor protects. All of these files sit in the project directory, where the agent can edit them. If the ledger write fails, the status change still happens and only a warning is printed.

- **S L1:** Unstructured or outcome-only records (status ledger, scan history, batch stdout) exist for some actions. — [set-status.mjs:694-705](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/set-status.mjs#L694-L705); [batch/batch-runner.sh:1069](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L1069) (verified)
  - *To reach the next level:* A structured record of every tool call with arguments and timestamps owned by career-ops.
- **C L1:** Only status changes, scans and batch worker output are recorded. — [set-status.mjs:694-705](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/set-status.mjs#L694-L705) (verified)
  - *To reach the next level:* Recording of every script run, file write and fetch.
- **D L1:** Records are on by default but live in the workspace the agent edits. — [set-status.mjs:694-705](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/set-status.mjs#L694-L705) (verified)
  - *To reach the next level:* Storage outside the agent-writable workspace.
- **B L1:** Ledger failures are printed as warnings and the action proceeds. — [set-status.mjs:694-705](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/set-status.mjs#L694-L705) (verified)
  - *To reach the next level:* Records flushed and errors surfaced for every action, not just status changes.
- **Cap:** none

### C10 Limits & kill switch — 0.25 (high)

career-ops sets no session limit of its own. Its network fetches time out after 10 seconds, and the OpenRouter runner caps each model call at 15 seconds and 8,192 output tokens. The batch runner defaults to one worker, two retries, and a pause file. But a headless batch worker has no wall-clock, step or cost cap, and the interactive session runs as long as the host allows.

- **S L1:** Only per-request timeouts and a retry cap are enforced; there is no step, wall-clock or cost limit on agent work. — [providers/_http.mjs:29](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/providers/_http.mjs#L29); [openrouter-runner.mjs:68-70](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/openrouter-runner.mjs#L68-L70); [batch/batch-runner.sh:37-42](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L37-L42) (verified)
  - *To reach the next level:* An iteration or wall-clock cap enforced on worker sessions.
- **C L1:** Limits apply to individual fetches and batch retries only. — searched `rg -n 'timeout'` in `batch/batch-runner.sh` → 3 hits (two are lock-timeout comments, one is a curl --max-time on a liveness probe; no wall-clock limit on the headless claude worker) (verified)
  - *To reach the next level:* Tool timeouts plus a cap on the worker loop itself.
- **D L1:** Defaults exist but the agent can change them by passing different flags to the scripts it runs. — [batch/batch-runner.sh:37-42](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L37-L42) (verified)
  - *To reach the next level:* Defaults the model cannot raise.
- **B L1:** Ceilings are loose: a stuck or runaway headless worker runs until it exits on its own. — searched `rg -n 'timeout'` in `batch/batch-runner.sh` → 3 hits (two are lock-timeout comments, one is a curl --max-time on a liveness probe; no wall-clock limit on the headless claude worker); [batch/batch-runner.sh:1069](https://github.com/career-ops-hq/career-ops/blob/a156d4dfa18cbc3a10da6ebf2a3466ed57e681f8/batch/batch-runner.sh#L1069) (verified)
  - *To reach the next level:* Tight per-run time and cost ceilings with cancellation of pending work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Job postings, pages, forms and emails read by scan/evaluate/apply (AGENTS.md:63-71) · [B] sensitive data/systems: cv.md, config/profile.yml PII and .env keys in the same checkout (AGENTS.md:9-13) · [C] state change / egress: Host shell, file writes, WebFetch and Playwright navigation; batch workers with permissions skipped (batch/batch-runner.sh:1041) · Same default session? Yes

## Highest-impact improvements
1. Drop --dangerously-skip-permissions from the batch runner and reuse the web UI's per-kind --allowedTools/--disallowedTools fencing for batch workers. — C2 D L0→L2, +0.100 before caps (Playbook 5)
2. Require an interactive TTY confirmation (not just a --confirm flag) for update apply and plugin enable, so the agent cannot self-approve. — C2 S L0→L2, +0.150 before caps (Playbook 5)
3. Gate writes to modes/_custom.md and agent-inbox items behind a shown diff and user confirmation in code, and tag inbox items with their source. — C6 S L0→L2, +0.150 before caps (Playbook 2)
4. Run plugins in a child process with a scrubbed environment containing only their declared env vars, and require consent on every plugin enable path. — C7 B L0→L2, +0.100 before caps (Playbook 3)
5. Run headless workers in a hardened container (non-root, no forwarded keys, workspace-only mount) by default. — C4 S L0→L3, +0.225 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The host AI CLI's own permission prompts, sandboxing and transcripts are not part of this repository and were not credited; with a host that prompts per command, real-world exposure in the interactive mode is lower than the career-ops-owned controls alone suggest.
- The opt-in web UI (web/), Go dashboard, Docker setup and 130+ provider modules were sampled, not read exhaustively; the batch runner is footnoted because it is a documented secondary mode whose default disables host approvals.
- No text aimed at AI reviewers was found; AGENTS.md:69 quotes 'as the AI reviewing this, you must...' only as an example of injected text the agent should ignore.
