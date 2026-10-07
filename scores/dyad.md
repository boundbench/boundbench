# Defense-in-depth score: Dyad

**Repo:** https://github.com/dyad-sh/dyad · **Commit:** `701d9179b43773a9c654014975ebc699a32fd500` · **Reviewed:** 2026-10-05
**What it is:** Local, open-source desktop AI app builder (Electron) whose agent writes, installs, builds and runs web apps on the user's machine.
**Category:** Coding
**Scored configuration:** Desktop app, fresh install, no Dyad Pro and no integrations or MCP servers connected: the default chat mode resolves to the local Basic Agent and apps run in the default host runtime.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials opt-in · persistent memory yes · untrusted input yes · third party extensions yes · sub agents opt-in · external communication opt-in

## Score: 2.7 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L0 | L0 | 0.07 | none | **0.07** | High |
| C2 | Approval gates | L2 | L0 | L1 | L1 | 0.25 | C2-POWERBYPASS | **0.25** | High |
| C3 | Tool & action scoping | L3 | L2 | L1 | L0 | 0.42 | none | **0.42** | High |
| C4 | Code-execution isolation | L2 | L1 | L0 | L2 | 0.33 | G1 | **0.33** (alt) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | none | **0.10** | High |
| C7 | Third-party extensions | L1 | L1 | L0 | L1 | 0.20 | C7-RCELOAD | **0.20** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L0 | 0.40 | none | **0.40** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | none | **0.50** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L1 | 0.45 | none | **0.45** | High |


Dyad's agent edits your app's files and then installs, builds and runs that app directly on your computer under your own account and full environment, and those steps need no approval by default. Only a few actions ask first: adding a package, schema-changing SQL and MCP tool calls. File tools are well contained to the app folder and .env values are redacted before they reach the model, but nothing contains the code the agent writes once the app runs, so a hijacked session can do anything you can.

## Critical gaps
- Code the agent writes runs on the host with the user's full account and environment when the app starts or builds, with no approval by default. (ASI02, ASI09; C2). Evidence: [src/pro/main/ipc/handlers/local_agent/tools/app_lifecycle.ts:46](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/app_lifecycle.ts#L46); [src/pro/main/ipc/handlers/local_agent/tools/run_build.ts:416](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/run_build.ts#L416)
- The app runtime, builds, tests and installs run unsandboxed on the host; only one scripting tool is sandboxed. (ASI05; C4). Evidence: [src/ipc/services/app_runtime_service.ts:889-897](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/services/app_runtime_service.ts#L889-L897); [src/pro/main/ipc/handlers/local_agent/tools/run_build.ts:420](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/run_build.ts#L420)
- A session hijacked by imported code, logs or MCP output can leak data and make irreversible changes with no human involved. (ASI01; C5). Evidence: [src/pro/main/ipc/handlers/local_agent/tools/write_file.ts:36](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/write_file.ts#L36); [src/ipc/services/app_runtime_service.ts:889-897](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/services/app_runtime_service.ts#L889-L897)
- Agent-started processes act with the user's whole account and inherited credentials. (ASI03; C1). Evidence: [src/ipc/services/app_runtime_service.ts:889](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/services/app_runtime_service.ts#L889)
- Third-party package code from the app's package.json is installed and executed without consent when the app runs or builds. (ASI04; C7). Evidence: [src/pro/main/ipc/handlers/local_agent/tools/run_build.ts:420](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/run_build.ts#L420)

## Criterion details

### C1 Identity & least privilege: 0.07 (high confidence)

The agent acts with the full authority of the logged-in user. The app's dev server, builds and package installs are started with a copy of Dyad's whole environment, so any credential in it is available to code the agent wrote. One path narrows this: end-to-end test runs strip the app's database credentials from the environment. Optional integrations add broad, long-lived credentials such as a Supabase management token and a GitHub token with repo and workflow scopes.

- **S L0:** No agent-specific identity: processes the agent starts run as the OS user with the inherited environment. Evidence: [src/ipc/services/app_runtime_service.ts:889-897](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/services/app_runtime_service.ts#L889-L897); [src/ipc/handlers/github_handlers.ts:108](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/handlers/github_handlers.ts#L108) (verified)
  - *To reach the next level:* No dedicated, narrowed identity or scrubbed credential set for the processes the agent starts.
- **C L1:** Test runs withhold database credentials, but the dev server, builds and installs inherit the full environment. Evidence: [src/ipc/handlers/tests_handlers.ts:2680-2685](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/handlers/tests_handlers.ts#L2680-L2685); [src/ipc/utils/socket_firewall.ts:150-154](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/utils/socket_firewall.ts#L150-L154) (verified)
  - *To reach the next level:* Every process the agent starts should receive only the credentials it needs.
- **D L0:** The default install runs everything with the user's full authority; there is no narrower default. Evidence: [src/ipc/services/app_runtime_service.ts:397](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/services/app_runtime_service.ts#L397) (verified)
  - *To reach the next level:* No reduced-privilege default for app runs, builds and installs.
- **B L0:** Code the agent writes runs on the host as the user, reaching the user's whole account, files and any credentials on the machine. Evidence: [src/ipc/services/app_runtime_service.ts:889-897](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/services/app_runtime_service.ts#L889-L897); [src/pro/main/ipc/handlers/local_agent/tools/run_build.ts:420](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/run_build.ts#L420) (verified)
  - *To reach the next level:* Agent-started processes should not hold the user's account-wide authority.
- **Cap:** none

### C2 Approval gates: 0.25 (high confidence)

Each agent tool has a consent level, and a handful default to asking: adding packages, schema-changing or deleting SQL, reinstalling dependencies and every MCP tool call. The prompt shows the exact SQL or package list, though MCP arguments are cut at 500 characters. Writing, deleting and renaming files, restarting the app, building and running tests all default to running without approval, and restarting or building executes code the agent just wrote. Each turn is committed to git, so file changes can be rolled back, but anything the running code does on the machine cannot.

- **S L2:** Per-call approval with per-tool risk defaults; previews show the exact SQL or packages, but MCP arguments are truncated. Evidence: [src/pro/main/ipc/handlers/local_agent/tool_definitions.ts:335-349](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tool_definitions.ts#L335-L349); [src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts:2929-2934](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts#L2929-L2934); [src/pro/main/ipc/handlers/local_agent/tools/execute_sql.ts:186](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/execute_sql.ts#L186) (verified)
  - *To reach the next level:* The approver does not always see the full call (MCP previews are truncated), and there is no argument-level policy.
- **C L0:** The most powerful path, writing code and then restarting or building the app, is exempt from approval by default. Evidence: [src/pro/main/ipc/handlers/local_agent/tools/write_file.ts:36](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/write_file.ts#L36); [src/pro/main/ipc/handlers/local_agent/tools/app_lifecycle.ts:46](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/app_lifecycle.ts#L46); [src/pro/main/ipc/handlers/local_agent/tools/run_build.ts:416](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/run_build.ts#L416); [src/pro/main/ipc/handlers/local_agent/tools/delete_file.ts:37](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/delete_file.ts#L37) (verified)
  - *To reach the next level:* File writes plus app restart, build and test runs should cross the gate.
- **D L1:** Approval is on by default only for a minority of tools, and non-schema SQL is auto-approved by a default-on setting. Evidence: [src/main/settings.ts:81](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/main/settings.ts#L81); [src/pro/main/ipc/handlers/local_agent/tool_definitions.ts:242-253](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tool_definitions.ts#L242-L253); [src/pro/main/ipc/handlers/local_agent/tools/add_dependency.ts:33](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/add_dependency.ts#L33) (verified)
  - *To reach the next level:* Consequential tools should ask by default, with auto-approve settings off unless the operator enables them.
- **B L1:** File changes are committed per turn and reversible, but code run on the host and changes to connected services are not. Evidence: [src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts:2376-2378](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts#L2376-L2378); [src/pro/main/ipc/handlers/local_agent/tools/run_build.ts:420](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/run_build.ts#L420) (verified)
  - *To reach the next level:* Actions outside the git working tree have no undo, preview or quantity bound.
- **Cap:** C2-POWERBYPASS: Writing code and restarting, building or testing the app runs that code on the host without approval in the default configuration (verified consent defaults).

### C3 Tool & action scoping: 0.42 (high confidence)

File tools resolve paths and symlinks and refuse anything outside the app folder, and reads open the resolved file and check it before reading. Package names are checked against strict npm name and version patterns, and outputs and file sizes are bounded. The weak points are reach rather than validation: by default the agent can write any file in the app and then run the app, build or tests, which executes whatever the app's scripts say. MCP tools get only schema validation.

- **S L3:** Resolved-path containment with symlink checks for writes and reads, plus allowlist-style package spec validation. Evidence: [src/ipc/utils/path_utils.ts:112-178](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/utils/path_utils.ts#L112-L178); [src/ipc/utils/bounded_text_file.ts:84-92](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/utils/bounded_text_file.ts#L84-L92); [src/ipc/processors/executeAddDependency.ts:101-108](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/processors/executeAddDependency.ts#L101-L108) (verified)
  - *To reach the next level:* Containment is check-then-use rather than race-proof, and general execution through app scripts remains.
- **C L2:** Built-in file, sandbox and package tools validate; MCP tools only pass JSON-schema validation. Evidence: [src/ipc/utils/sandbox/capabilities.ts:205](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/utils/sandbox/capabilities.ts#L205); [src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts:2923-2927](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts#L2923-L2927) (verified)
  - *To reach the next level:* No shared allowlist validation layer for extension tools.
- **D L1:** Write, delete and app-run tools are on by default; each tool can be disabled individually by setting it to never. Evidence: [src/pro/main/ipc/handlers/local_agent/tool_definitions.ts:741-743](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tool_definitions.ts#L741-L743); [src/pro/main/ipc/handlers/local_agent/tool_definitions.ts:148-155](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tool_definitions.ts#L148-L155) (verified)
  - *To reach the next level:* A read-only default tool set with write and execute tools enabled explicitly.
- **B L0:** Through the app run and build tools the agent reaches arbitrary commands on the whole machine. Evidence: [src/pro/main/ipc/handlers/local_agent/tools/run_build.ts:420](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/run_build.ts#L420) (verified)
  - *To reach the next level:* Execution reach should be confined to the app workspace.
- **Cap:** none

### C4 Code-execution isolation: 0.33 (high confidence)

The only sandbox on by default is a small scripting runtime used by one tool, which gets file and MCP access only through functions Dyad injects and has time and memory limits. Everything else that runs code does so directly on the host: the app's dev server, production builds, type checks, tests and package installs, all under the user's account with the inherited environment. An optional Docker runtime runs the dev server in a stock container with the app folder mounted and normal network access, but builds and installs still run on the host.

- **default configuration** (default; raw 0.20 → 0.20)
  - **S L2:** The default sandbox is the MustardScript capability runtime; its isolation properties come from a third-party library not present in the repository. Evidence: [src/ipc/utils/sandbox/execution.ts:70](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/utils/sandbox/execution.ts#L70); [src/ipc/utils/sandbox/limits.ts:7-14](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/utils/sandbox/limits.ts#L7-L14) (inferred)
    - *To reach the next level:* A verified capability runtime or OS-level sandbox for the main execution paths.
  - **C L0:** Only the sandbox-script tool is contained; the app runtime, builds and installs run on the host. Evidence: [src/ipc/services/app_runtime_service.ts:889-897](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/services/app_runtime_service.ts#L889-L897); [src/pro/main/ipc/handlers/local_agent/tools/run_build.ts:420](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/run_build.ts#L420) (verified)
    - *To reach the next level:* The app runtime, build, test and install paths should go through the sandbox.
  - **D L1:** The script sandbox is on by default, but host execution is the default runtime for everything else. Evidence: [src/main/settings.ts:69](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/main/settings.ts#L69); [src/ipc/services/app_runtime_service.ts:397](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/services/app_runtime_service.ts#L397) (verified)
    - *To reach the next level:* Isolation should be the default for every execution path, disabled only by an explicit operator flag.
  - **B L0:** Host-equivalent: unsandboxed processes run as the user with the full environment, home directory and network. Evidence: [src/ipc/services/app_runtime_service.ts:889-897](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/services/app_runtime_service.ts#L889-L897) (verified)
    - *To reach the next level:* Workspace-only access, no secrets in the environment and restricted egress for executed code.
- **opt-in Docker runtime** (alt; raw 0.33, cap G1 → 0.33) ← counted
  - **S L2:** Stock node:22-alpine container with default user and capabilities. Evidence: [src/ipc/services/app_runtime_service.ts:1367](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/services/app_runtime_service.ts#L1367); [src/ipc/services/app_runtime_service.ts:1412-1426](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/services/app_runtime_service.ts#L1412-L1426) (verified)
    - *To reach the next level:* Non-root user, dropped capabilities, no-new-privileges and seccomp.
  - **C L1:** Only the dev server runs in the container; builds still spawn on the host. Evidence: [src/pro/main/ipc/handlers/local_agent/tools/run_build.ts:359-363](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/run_build.ts#L359-L363) (verified)
    - *To reach the next level:* Builds, tests, type checks and installs should also run in the container.
  - **D L0:** The Docker runtime is opt-in; host is the default. Evidence: [src/ipc/services/app_runtime_service.ts:397](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/services/app_runtime_service.ts#L397) (verified)
    - *To reach the next level:* Containerised execution on by default.
  - **B L2:** The app folder is mounted read-write and the container has default network access, without the host environment. Evidence: [src/ipc/services/app_runtime_service.ts:1421-1426](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/services/app_runtime_service.ts#L1421-L1426) (verified)
    - *To reach the next level:* Egress restriction and resource limits.
- **Cap:** G1: Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius: 0.00 (high confidence)

The agent reads content its user did not write, such as imported repositories, app console logs and MCP tool results, and nothing in code separates that content from the user's instructions. There is no taint tracking and no rule that forces approval once untrusted content has been read; the only defence is prompt wording. A hijacked session can write code and run it on the host without approval, which is enough both to send data anywhere and to make irreversible changes.

- **S L0:** No structural limit; untrusted-content handling is prompt text only. Evidence: searched `rg -n -i -e 'taint|untrusted|quarantin|provenance'` in `src/pro/main/ipc/handlers/local_agent/tool_definitions.ts src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts` → 1 hits (The one hit is a prompt instruction to treat Pro sub-agent reports as untrusted; nothing enforces it.) (verified)
  - *To reach the next level:* Approval or capability removal once untrusted content enters the session.
- **C L0:** Tool results, file contents and MCP outputs enter context with the same standing as user input. Evidence: [src/pro/main/ipc/handlers/local_agent/tool_definitions.ts:423-428](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tool_definitions.ts#L423-L428) (verified)
  - *To reach the next level:* Untrusted sources should be distinguished and covered by the limit.
- **D L0:** No untrusted-input control exists to be on by default. Evidence: searched `rg -n -i -e 'taint|untrusted|quarantin|provenance'` in `src/pro/main/ipc/handlers/local_agent/tool_definitions.ts src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts` → 1 hits (Prompt text only.) (verified)
  - *To reach the next level:* A default-on control that content cannot disable.
- **B L0:** A hijacked agent can write code and run it on the host unattended, so it can both exfiltrate data and take irreversible actions. Evidence: [src/pro/main/ipc/handlers/local_agent/tools/write_file.ts:36](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/write_file.ts#L36); [src/pro/main/ipc/handlers/local_agent/tools/app_lifecycle.ts:46](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/app_lifecycle.ts#L46); [src/ipc/services/app_runtime_service.ts:889-897](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/services/app_runtime_service.ts#L889-L897) (verified)
  - *To reach the next level:* Egress and host execution should need a human once untrusted content is read.
- **Cap:** C5-WORSTCASE: B is L0: unattended host code execution gives both an exfiltration channel and irreversible actions in the default configuration.

### C6 Memory, context & configuration integrity: 0.10 (high confidence)

Each app's AI_RULES.md is read from the app folder on every turn and placed in the system prompt as authoritative project guidance, and the agent can edit that file without approval. An imported repository can therefore ship its own rules, and anything written there steers every later session in that app. Chat history from earlier conversations is also searchable by the agent. The file is versioned in the app's git history, so changes can be inspected and reverted by hand.

- **S L0:** The model can write AI_RULES.md freely and it is re-injected as authoritative context; repository copies load silently. Evidence: [src/prompts/system_prompt.ts:916-919](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/prompts/system_prompt.ts#L916-L919); [src/prompts/local_agent_prompt.ts:424-425](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/prompts/local_agent_prompt.ts#L424-L425) (verified)
  - *To reach the next level:* Changes to persistent instruction files should be gated or reviewed, and repository-supplied rules need a trust decision.
- **C L0:** Neither the rules file nor chat history recall is controlled. Evidence: [src/pro/main/ipc/handlers/local_agent/tool_definitions.ts:166-168](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tool_definitions.ts#L166-L168) (verified)
  - *To reach the next level:* At least the main persistent store should be validated or gated.
- **D L1:** Rules are scoped per app folder on a single-user desktop, but nothing beyond the folder layout enforces it. Evidence: [src/prompts/system_prompt.ts:916-919](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/prompts/system_prompt.ts#L916-L919) (verified)
  - *To reach the next level:* Enforced per-app namespaces the model cannot change.
- **B L1:** Poisoned rules persist across the user's sessions for that app and can drive ungated tool use. Evidence: [src/prompts/local_agent_prompt.ts:424-425](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/prompts/local_agent_prompt.ts#L424-L425) (verified)
  - *To reach the next level:* Persistent context should only influence gated actions or require review before reuse.
- **Cap:** none
- **Notes:** Dyad's main process calls dotenv.config() at startup (src/main.ts:389), which reads a .env from the launch directory; for a desktop launch this is not the app workspace, so C6-REPOCONFIG was not applied.

### C7 Third-party extensions: 0.20 (high confidence)

MCP servers are added by the user (a deep link only pre-fills a dialog) and run whatever command and arguments were entered, unpinned. Package installs through the dedicated tool ask first and are wrapped in a pinned Socket firewall that blocks known-malicious packages, falling back to an unscreened install with a warning when it can't be set up. Running or building the app installs and executes the packages its package.json lists without a prompt, as the build tool's own description acknowledges. Installed packages and MCP servers run as the user.

- **S L1:** User-chosen, unpinned MCP commands; packages screened by a malware firewall but not pinned or hash-checked. Evidence: [src/ipc/utils/mcp_manager.ts:172-184](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/utils/mcp_manager.ts#L172-L184); [src/ipc/utils/socket_firewall.ts:45](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/utils/socket_firewall.ts#L45) (verified)
  - *To reach the next level:* Pinned versions with integrity checks for MCP servers and agent-installed packages.
- **C L1:** A malware screen exists for package installs; MCP servers are not verified at all. Evidence: [src/ipc/utils/socket_firewall.ts:1315-1328](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/utils/socket_firewall.ts#L1315-L1328); searched `rg -n -S -e 'sha256|integrity|checksum|signature'` in `src/ipc/utils/mcp_manager.ts src/ipc/handlers/mcp_handlers.ts` → 0 hits (No integrity check on MCP server launch.) (verified)
  - *To reach the next level:* Pinned, integrity-checked installs for every extension type.
- **D L0:** Dependencies listed in the app's package.json, which the agent and imported repositories can edit, are installed and run without a consent prompt. Evidence: [src/pro/main/ipc/handlers/local_agent/tools/run_build.ts:420](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/run_build.ts#L420); [src/pro/main/ipc/handlers/local_agent/tools/write_file.ts:36](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/write_file.ts#L36) (verified)
  - *To reach the next level:* Workspace content should not be able to add executed third-party code silently.
- **B L1:** Package scripts run in a separate process as the same user with the full environment. Evidence: [src/ipc/utils/socket_firewall.ts:150-154](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/utils/socket_firewall.ts#L150-L154) (verified)
  - *To reach the next level:* Scrubbed environment and per-extension sandboxing.
- **Cap:** C7-RCELOAD: By default, running or building the app installs and executes third-party package code from the app's package.json without consent; the run-build tool's own preview acknowledges this (verified).

### C8 Secrets & sensitive-data protection: 0.40 (high confidence)

Provider keys and integration tokens are encrypted with the operating system's secure storage, falling back to plaintext only when it is unavailable. Values in .env files are redacted before file reads, grep, sandbox scripts and codebase context reach the model, and model-request logs record only metadata. Telemetry is opt-in. The gaps are the environment handed to app, build and install processes, unredacted build and log output, and chat transcripts stored unencrypted.

- **S L2:** OS secure-storage encryption and .env redaction on the main model-bound paths, with a plaintext fallback and unredacted command output. Evidence: [src/main/settings.ts:1141-1149](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/main/settings.ts#L1141-L1149); [src/pro/main/ipc/handlers/local_agent/tools/read_file.ts:160-163](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/read_file.ts#L160-L163); [src/utils/codebase.ts:94](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/utils/codebase.ts#L94) (verified)
  - *To reach the next level:* Redaction on every model-bound and log path, including build and runtime output, and no plaintext fallback.
- **C L2:** Logs and the main model-bound reads are covered; subprocess environments and transcripts are not. Evidence: [src/ipc/utils/model_request_logging.ts:8-13](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/utils/model_request_logging.ts#L8-L13); [src/ipc/services/app_runtime_service.ts:889](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/services/app_runtime_service.ts#L889) (verified)
  - *To reach the next level:* Subprocess environments and stored transcripts should also be protected.
- **D L2:** Telemetry is sent only after an explicit opt-in; redaction is always on for the covered paths. Evidence: [src/renderer.tsx:121-126](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/renderer.tsx#L121-L126); [src/main/settings.ts:54](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/main/settings.ts#L54) (verified)
  - *To reach the next level:* Content-free telemetry and minimised or encrypted stored transcripts.
- **B L0:** Every agent-started process inherits the full environment and runs as the user, so long-lived keys on the machine are reachable. Evidence: [src/ipc/utils/socket_firewall.ts:150-154](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/utils/socket_firewall.ts#L150-L154) (verified)
  - *To reach the next level:* Scoped, short-lived credentials kept out of agent-started processes.
- **Cap:** none

### C9 Audit & traceability: 0.50 (high confidence)

Every tool call is written into the chat message in Dyad's local database as it happens, including its arguments and result, and the full model message history is saved at the end of the turn. Each turn is also committed to the app's git repository. The record carries no separate actor or approver fields, approvals and denials are not clearly logged, and the database sits in the user's data folder where code running as the user could change it.

- **S L2:** Structured local transcript of tool calls and results per message. Evidence: [src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts:732-741](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts#L732-L741) (verified)
  - *To reach the next level:* Actor and approver attribution with correlation IDs.
- **C L2:** All built-in and MCP tool calls pass through the same recording wrapper; approval decisions are not recorded. Evidence: [src/pro/main/ipc/handlers/local_agent/tool_definitions.ts:1038-1039](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tool_definitions.ts#L1038-L1039) (verified)
  - *To reach the next level:* Record approvals and denials as well as calls.
- **D L2:** On by default and stored outside the app workspace, but writable by any process running as the user. Evidence: [src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts:2376-2378](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts#L2376-L2378) (verified)
  - *To reach the next level:* Written by a component the model's code cannot reach.
- **B L2:** Records are flushed per tool call; logging failure does not stop actions. Evidence: [src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts:741](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts#L741) (verified)
  - *To reach the next level:* Durable, replayable per-action records.
- **Cap:** none

### C10 Limits & kill switch: 0.45 (high confidence)

A turn stops after 100 tool steps by default (selectable from 25 to 200), builds are limited to three per turn, and builds, installs, shell commands and sandbox scripts have timeouts. There is no token or cost cap for a run beyond the free tier's message quota. Stopping a turn aborts the model stream and passes a cancel signal to tools, but the app's dev server keeps running, and code the agent runs on the host can modify Dyad's own settings.

- **S L2:** Step cap plus per-tool timeouts, enforced in code. Evidence: [src/constants/settings_constants.ts:2](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/constants/settings_constants.ts#L2); [src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts:1455](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts#L1455); [src/pro/main/ipc/handlers/local_agent/tools/run_build.ts:41-42](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/run_build.ts#L41-L42) (verified)
  - *To reach the next level:* A token or cost cap and a run-level time bound.
- **C L2:** Top-level loop plus tool timeouts; background app processes are outside the budget. Evidence: [src/ipc/utils/sandbox/limits.ts:7](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/ipc/utils/sandbox/limits.ts#L7) (verified)
  - *To reach the next level:* Spawned processes should count against the same budget.
- **D L2:** Sensible default, operator-configurable with no hard ceiling in the schema. Evidence: [src/lib/schemas.ts:511](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/lib/schemas.ts#L511); [src/components/MaxToolCallStepsSelector.tsx:41](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/components/MaxToolCallStepsSelector.tsx#L41) (verified)
  - *To reach the next level:* Limits the agent's own processes cannot raise, with a hard ceiling.
- **B L1:** 100 steps with multi-minute tool timeouts and no spend cap; the dev server outlives a stop. Evidence: [src/pro/main/ipc/handlers/local_agent/tools/run_build.ts:42](https://github.com/dyad-sh/dyad/blob/701d9179b43773a9c654014975ebc699a32fd500/src/pro/main/ipc/handlers/local_agent/tools/run_build.ts#L42) (verified)
  - *To reach the next level:* Tight time and cost ceilings and no processes left running after a stop.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: imported repositories and app files, app console logs, MCP results (tool_definitions.ts:423-428) · [B] sensitive data/systems: workspace code, chat history, the user's home directory and environment (app_runtime_service.ts:889) · [C] state change / egress: unapproved file writes plus host execution via app restart and build (write_file.ts:36, app_lifecycle.ts:46) · Same default session? Yes

## Highest-impact improvements
1. Default app restart, build, test and type-check tools to ask, showing what will execute. (C2 C L0→L2, +0.150 before caps; Playbook 5)
2. Run the app, builds, tests and installs in a hardened container by default (non-root, dropped capabilities, egress restricted). (C4 C L0→L2, +0.150 before caps; Playbook 3)
3. Pass only an allowlisted environment to app, build and install processes. (C8 C L2→L3, +0.075 before caps; Playbook 4)
4. Once imported code, logs or MCP output are in the session, require approval for host execution and network-reaching tools. (C5 S L0→L2, +0.150 before caps; Playbook 1)
5. Show a diff and ask before AI_RULES.md changes take effect, and ask before trusting rules from imported repositories. (C6 S L0→L2, +0.150 before caps; Playbook 2)

## Re-audit log
- C2 S: L3 → L2. Anchor recheck: MCP consent previews are truncated to 500 characters (local_agent_handler.ts:2934), so the approver does not always see the exact call.
- C8 S: L3 → L2. Attacked L3: secure storage falls back to plaintext (settings.ts:1149) and build/runtime output reaching the model is not redacted.
- C10 B: L2 → L1. No spend cap and up to 100 steps with 10-minute tool timeouts; the dev server continues after a stop.
- C4 S: L3 → L2. MustardScript's isolation is a property of a third-party package not in the repository, so it is inferred and capped at L2.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the non-Pro Basic Agent default. Pro-only features (web fetch/search, the opt-in run_shell tool with an LLM safety reviewer, sub-agents, LLM auto-approval of MCP calls) and the experimental cloud runtime were not scored and are noted only where relevant.
- The MustardScript runtime and the pg-schema-classifier SQL classifier were not reviewed in depth; their behaviour is inferred from how Dyad calls them.
- The legacy Build chat mode, the preview window and deployment integrations (Supabase, Neon, Vercel, GitHub) were reviewed only where the agent loop reaches them.
