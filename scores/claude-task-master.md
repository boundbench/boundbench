# Defense-in-depth score: Task Master (claude-task-master)

**Repo:** https://github.com/eyaltoledano/claude-task-master · **Commit:** `c0c98d367c55296bfe69e65680625b6db437af02` · **Reviewed:** 2026-10-05
**What it is:** AI task-management system for coding agents, used as an MCP server in editors such as Cursor and Claude Code or as a CLI.
**Category:** Coding
**Scored configuration:** MCP server over stdio launched with `npx -y task-master-ai` as the README recommends, TASK_MASTER_TOOLS unset (all tools), default models (Anthropic main, Perplexity research), local file storage, telemetry default.
**Agent surface (default):** code execution opt-in · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication no

## Score: 2.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | none | **0.05** | High |
| C2 | Approval gates | L2 | L2 | L2 | L1 | 0.45 | none | **0.45** | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | none | **0.45** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L1 | 0.12 | none | **0.12** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | L1 | L0 | L0 | L0 | 0.07 | none | **0.07** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L0 | L1 | 0.20 | none | **0.20** | High |
| C9 | Audit & traceability | L1 | L1 | L1 | L1 | 0.25 | none | **0.25** | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | none | **0.33** | High |


Task Master runs with your full user authority and every provider API key you give it, and it loads project configuration and .env files from the workspace without asking, so an untrusted repository can change which model provider and endpoints it uses. Telemetry is on by default and records prompts and responses. Tool annotations are accurate, but there is no path containment, isolation or audit record.

## Critical gaps
- Workspace configuration and .env files can change model providers, endpoints and provider settings with no trust decision. (ASI06, ASI04; C6). Evidence: [scripts/modules/config-manager.js:136-144](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/config-manager.js#L136-L144); [apps/mcp/src/shared/utils.ts:401-403](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/apps/mcp/src/shared/utils.ts#L401-L403)
- Agent-CLI providers run on the host as the same user with the full environment and no isolation. (ASI05; C4). Evidence: [packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts:124-130](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts#L124-L130)
- Launched provider CLIs are unverified and receive every provider key in the environment. (ASI04; C7). Evidence: [packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts:124-130](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts#L124-L130)

## Criterion details

### C1 Identity & least privilege: 0.05 (high confidence)

Task Master runs as a local stdio process with the operator's full user authority and whatever provider API keys are placed in the MCP environment or a project .env file. Nothing narrows that authority: tools accept any absolute project directory, and spawned provider CLIs inherit the whole environment. There is no per-tool or per-request credential scoping. The keys it holds are LLM-provider keys, so the damage is mostly spend and local file changes rather than access to production systems.

- **S L0:** Ambient OS-user authority plus every provider key in the environment; no narrowing of either. Evidence: [mcp-server/src/index.js:121-124](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/index.js#L121-L124); [packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts:124-130](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts#L124-L130) (verified)
  - *To reach the next level:* Use dedicated, narrowly scoped credentials per provider role and refuse unrelated environment credentials.
- **C L0:** No authorization check runs before any tool; the model-supplied projectRoot decides where tools act. Evidence: [apps/mcp/src/shared/utils.ts:298](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/apps/mcp/src/shared/utils.ts#L298); [mcp-server/src/tools/tool-registry.js:59-104](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/tools/tool-registry.js#L59-L104) (verified)
  - *To reach the next level:* Put every tool behind one authorization layer that limits it to an approved project directory.
- **D L0:** The default launch exposes all tools with the user's full authority and all configured keys. Evidence: [README.md:301](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/README.md#L301); [README.md:133](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/README.md#L133) (verified)
  - *To reach the next level:* Ship a minimal default (read-only tools, keys scoped to the selected provider).
- **B L1:** A hijacked session can spend on every configured provider account and write task/config files anywhere the user can. Evidence: [mcp-server/src/core/direct-functions/parse-prd.js:84-87](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/core/direct-functions/parse-prd.js#L84-L87); [mcp-server/src/tools/models.js:20-25](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/tools/models.js#L20-L25) (verified)
  - *To reach the next level:* Limit reach to one project directory and one provider credential.
- **Cap:** none

### C2 Approval gates: 0.45 (high confidence)

As an MCP tool server, Task Master leaves approval to the host but labels its tools: every tool carries a read-only or destructive hint, and reads and writes are separate tools. There are no previews or dry runs for destructive operations and no server-enforced read-only mode. Writes are mostly to task files, but some tools overwrite files at caller-chosen paths or create git commits of all changes, with no undo of their own.

- **S L2:** Separate read and write tools, each annotated with readOnlyHint or destructiveHint. Evidence: [mcp-server/src/tools/parse-prd.js:68-71](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/tools/parse-prd.js#L68-L71); [mcp-server/src/tools/next-task.js:43](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/tools/next-task.js#L43); [apps/mcp/src/tools/autopilot/commit.tool.ts:39-42](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/apps/mcp/src/tools/autopilot/commit.tool.ts#L39-L42) (verified)
  - *To reach the next level:* Offer a preview or dry-run for destructive operations such as remove_task, delete_tag and parse_prd overwrite.
- **C L2:** Hints cover every registered tool, but when an agent-CLI provider is selected its actions happen inside a tool call and never reach the host's gate. Evidence: [mcp-server/src/tools/tool-registry.js:59-104](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/tools/tool-registry.js#L59-L104); [scripts/modules/task-manager/models.js:685](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/task-manager/models.js#L685) (verified)
  - *To reach the next level:* Keep all consequential actions inside annotated tool calls, including those taken by provider agents.
- **D L2:** Hints are fixed in code, but there is no server-side read-only mode and the full write-capable tool set loads by default. Evidence: [README.md:301](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/README.md#L301); [mcp-server/src/tools/remove-task.js:34](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/tools/remove-task.js#L34) (verified)
  - *To reach the next level:* Ship a read-only mode or a server-side confirmation the host must complete for destructive tools.
- **B L1:** A wrongly approved call can overwrite files at caller-chosen paths, delete task data or commit every working-tree change; the server keeps no backup or undo. Evidence: [mcp-server/src/core/direct-functions/parse-prd.js:84-87](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/core/direct-functions/parse-prd.js#L84-L87); [apps/mcp/src/tools/autopilot/commit.tool.ts:107](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/apps/mcp/src/tools/autopilot/commit.tool.ts#L107) (verified)
  - *To reach the next level:* Add backups or rollback for task data and bound writes to the project's task files.
- **Cap:** none

### C3 Tool & action scoping: 0.45 (high confidence)

Tools are narrow task-management operations with typed zod schemas, which keeps designed use small. Path arguments are not contained: the project root, research file paths and the parse_prd output path all accept any absolute path. The default loads all tools, including write, git and network-backed research tools, though tool groups can be selected with an environment variable.

- **S L2:** Typed schemas and narrow task tools, with a 50KB file-size cap, but file paths are taken as given. Evidence: [scripts/modules/utils/contextGatherer.js:646-648](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/utils/contextGatherer.js#L646-L648); [scripts/modules/utils/contextGatherer.js:660](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/utils/contextGatherer.js#L660); [mcp-server/src/core/direct-functions/parse-prd.js:84-87](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/core/direct-functions/parse-prd.js#L84-L87) (verified)
  - *To reach the next level:* Resolve every path and require it to stay inside an approved project directory.
- **C L2:** All built-in tools use typed schemas; none applies path containment. Evidence: [mcp-server/src/tools/tool-registry.js:59-104](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/tools/tool-registry.js#L59-L104); [apps/mcp/src/shared/utils.ts:298](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/apps/mcp/src/shared/utils.ts#L298) (verified)
  - *To reach the next level:* Apply one shared path and argument policy to every tool.
- **D L2:** Tool groups (core, standard, custom) are selectable, but the default is all tools. Evidence: [README.md:301](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/README.md#L301) (verified)
  - *To reach the next level:* Default to a read-only tool set and require explicit enabling of write and network tools.
- **B L1:** A misused tool reaches any file the user can read (under 50KB) or write task data to any path. Evidence: [scripts/modules/utils/contextGatherer.js:646-648](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/utils/contextGatherer.js#L646-L648); [mcp-server/src/core/direct-functions/parse-prd.js:84-87](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/core/direct-functions/parse-prd.js#L84-L87) (verified)
  - *To reach the next level:* Scope tools to the project directory and bound quantities.
- **Cap:** none

### C4 Code-execution isolation: 0.00 (high confidence)

With the default Anthropic API provider, Task Master runs no model-written code. It also ships providers that launch coding-agent CLIs (Claude Code, Codex, Gemini CLI, Grok CLI), and the models tool lets the host model switch to them. Those CLIs run on the host as the same user with the full environment, including every API key, and Task Master adds no isolation around them.

- **S L0:** Agent-CLI providers run as same-user host processes; Task Master provides no isolation primitive. Evidence: [packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts:124-130](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts#L124-L130); [src/ai-providers/claude-code.js:5](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/src/ai-providers/claude-code.js#L5); searched `rg -n -S -e 'sandbox|seccomp|bwrap|landlock'` in `mcp-server/src apps/mcp/src src packages/ai-sdk-provider-grok-cli/src` → 0 hits (no isolation primitive in the MCP server or provider launch code) (verified)
  - *To reach the next level:* Run agent-CLI providers in a container or OS sandbox.
- **C L0:** No execution path is isolated. Evidence: [packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts:124-130](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts#L124-L130); searched `rg -n -S -e 'sandbox|seccomp|bwrap|landlock'` in `mcp-server/src apps/mcp/src src packages/ai-sdk-provider-grok-cli/src` → 0 hits (no isolation primitive in the MCP server or provider launch code) (verified)
  - *To reach the next level:* Route every provider subprocess through the sandbox.
- **D L0:** No isolation exists to turn on, and the models tool can select an agent-CLI provider at runtime. Evidence: [mcp-server/src/tools/models.js:20-25](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/tools/models.js#L20-L25); [scripts/modules/task-manager/models.js:685](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/task-manager/models.js#L685) (verified)
  - *To reach the next level:* Ship a sandbox on by default and require operator consent to select agent-CLI providers.
- **B L0:** A launched agent CLI holds the user's files, network and the full environment with every provider key. Evidence: [packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts:124-130](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts#L124-L130) (verified)
  - *To reach the next level:* Scrub the environment and confine filesystem and network access for provider subprocesses.
- **Cap:** none

### C5 Untrusted input blast radius: 0.12 (high confidence)

Task Master turns PRDs, project files and Perplexity web research into tasks that the host coding agent then reads and follows, with nothing marking which text came from untrusted sources. Responses wrap data in a JSON envelope with version and tag metadata, but task content carries no provenance. Assume a hijacked host: it can send any readable file to the research provider or point model calls at another endpoint through the models tool, but destructive changes are limited to task data and git commits.

- **S L1:** JSON envelope separates data from metadata, but task and research content has no provenance and tool descriptions carry directives to the agent. Evidence: [apps/mcp/src/shared/utils.ts:177-180](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/apps/mcp/src/shared/utils.ts#L177-L180); searched `rg -n -S -e 'untrusted|provenance|injection'` in `mcp-server/src apps/mcp/src src` → 0 hits (no untrusted-content marking or provenance anywhere in the server or provider code) (verified)
  - *To reach the next level:* Mark content derived from PRDs, files and web research with its source and an untrusted flag.
- **C L0:** No untrusted source is distinguished. Evidence: searched `rg -n -S -e 'untrusted|provenance|injection'` in `mcp-server/src apps/mcp/src src` → 0 hits (no untrusted-content marking or provenance anywhere in the server or provider code) (verified)
  - *To reach the next level:* Tag every untrusted source, including research results and stored task text.
- **D L0:** No mechanism exists, and there is no mode that drops the egress or write leg. Evidence: [README.md:301](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/README.md#L301); [mcp-server/src/tools/research.js:72](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/tools/research.js#L72) (verified)
  - *To reach the next level:* Offer read-only and no-egress modes and keep them on by default.
- **B L1:** A hijacked host can send local files to the research provider or another configured endpoint unattended; destructive actions are limited to task data and reversible git commits. Evidence: [mcp-server/src/tools/research.js:32](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/tools/research.js#L32); [scripts/modules/utils/contextGatherer.js:646-648](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/utils/contextGatherer.js#L646-L648); [mcp-server/src/tools/models.js:20-25](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/tools/models.js#L20-L25) (verified)
  - *To reach the next level:* Remove unattended egress of local files.
- **Cap:** none

### C6 Memory, context & configuration integrity: 0.10 (high confidence)

Task Master keeps tasks, research notes and settings as project files that the host agent reads back in later sessions, and the task text it stores is built from model output, PRDs and web research. It loads the project's .taskmaster configuration and .env files from the workspace on every call without any trust decision, and those files choose the model provider and security-relevant endpoints and settings. Nothing validates or tags what is persisted.

- **S L0:** Workspace configuration and .env files are loaded silently and can change providers, endpoints and settings; stored task text is re-served without validation. Evidence: [scripts/modules/config-manager.js:136-144](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/config-manager.js#L136-L144); [apps/mcp/src/shared/utils.ts:401-403](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/apps/mcp/src/shared/utils.ts#L401-L403); [mcp-server/src/index.js:17](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/index.js#L17) (verified)
  - *To reach the next level:* Require a workspace-trust decision before project files can change providers, endpoints or provider settings.
- **C L0:** No persistence or auto-loaded config path is controlled. Evidence: [scripts/modules/config-manager.js:136-144](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/config-manager.js#L136-L144); [apps/mcp/src/shared/utils.ts:401-403](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/apps/mcp/src/shared/utils.ts#L401-L403) (verified)
  - *To reach the next level:* Control every auto-loaded file and persisted store.
- **D L1:** Data is per project by file location only; there is no isolation control beyond that. Evidence: [scripts/modules/config-manager.js:136-144](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/config-manager.js#L136-L144) (verified)
  - *To reach the next level:* Keep security settings in user scope and isolate stored data by default.
- **B L1:** Poisoned tasks or config persist across the user's sessions in that project and steer the host agent's tool use. Evidence: [apps/mcp/src/shared/utils.ts:401-403](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/apps/mcp/src/shared/utils.ts#L401-L403); [scripts/modules/config-manager.js:136-144](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/config-manager.js#L136-L144) (verified)
  - *To reach the next level:* Make persisted task text reviewable before use and easy to roll back.
- **Cap:** C6-REPOCONFIG: Workspace configuration and .env files can change the model provider, security-relevant endpoints and provider settings with no trust decision.

### C7 Third-party extensions: 0.07 (high confidence)

Task Master does not install plugins, but its agent-CLI providers launch whatever claude, codex, gemini or grok binary is on the user's PATH, with no version pinning or verification. Project configuration or the models tool can switch to these providers without a separate consent step. The launched CLI runs as the same user with the full environment.

- **S L1:** User-installed agent CLIs are launched from PATH, unpinned and unverified. Evidence: [packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts:124-130](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts#L124-L130); [src/ai-providers/claude-code.js:5](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/src/ai-providers/claude-code.js#L5) (verified)
  - *To reach the next level:* Pin and verify the provider CLIs the server launches.
- **C L0:** No launched CLI is verified. Evidence: [packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts:124-130](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts#L124-L130) (verified)
  - *To reach the next level:* Verify every launched provider CLI.
- **D L0:** Workspace configuration or the models tool can switch to an agent-CLI provider with no consent step. Evidence: [scripts/modules/config-manager.js:136-144](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/config-manager.js#L136-L144); [scripts/modules/task-manager/models.js:685](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/task-manager/models.js#L685) (verified)
  - *To reach the next level:* Only let user-scope configuration enable agent-CLI providers, after showing what will run.
- **B L0:** The launched CLI gets the full process environment, including every provider key. Evidence: [packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts:124-130](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts#L124-L130) (verified)
  - *To reach the next level:* Scrub the environment passed to provider CLIs.
- **Cap:** none

### C8 Secrets & sensitive-data protection: 0.20 (high confidence)

Provider keys come from the MCP environment or a project .env file and are held in process memory. Anonymous telemetry to Sentry is on by default and, as the changelog documents, records AI prompts and responses and MCP tool interactions, with default PII sending enabled. Error-message redaction exists only on the CLI path. A stored Hamster session, if the user logs in, is written with owner-only permissions.

- **S L1:** Keys come from env vars; a redaction pattern list exists but only the CLI error path uses it. Evidence: [scripts/modules/error-formatter.js:27](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/error-formatter.js#L27); [scripts/modules/commands.js:92](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/commands.js#L92); [packages/tm-core/src/modules/auth/services/context-store.ts:98](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/tm-core/src/modules/auth/services/context-store.ts#L98) (verified)
  - *To reach the next level:* Redact secrets before logs, telemetry and model-bound messages on all paths.
- **C L1:** Only CLI error messages are redacted; telemetry, MCP responses and subprocess environments are not. Evidence: [scripts/modules/commands.js:92](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/commands.js#L92); [packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts:124-130](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts#L124-L130) (verified)
  - *To reach the next level:* Cover telemetry, logs, tool responses and subprocess environments.
- **D L0:** Telemetry is on by default and records prompts and responses to a third party. Evidence: [scripts/modules/config-manager.js:61](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/config-manager.js#L61); [scripts/modules/config-manager.js:747](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/config-manager.js#L747); [src/telemetry/sentry.js:82-91](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/src/telemetry/sentry.js#L82-L91); [CHANGELOG.md:549](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/CHANGELOG.md#L549) (verified)
  - *To reach the next level:* Make telemetry opt-in and content-free.
- **B L1:** Credentials at stake are long-lived provider API keys, alongside project content sent in prompts. Evidence: [packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts:124-130](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts#L124-L130); [src/telemetry/sentry.js:83](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/src/telemetry/sentry.js#L83) (verified)
  - *To reach the next level:* Use scoped, short-lived credentials.
- **Cap:** none

### C9 Audit & traceability: 0.25 (high confidence)

Task Master writes no audit record of its own for most tool calls; it sends log lines to the host over MCP and to stderr. The autopilot workflow appends structured events to an activity.jsonl file under the user's home directory, which covers only that workflow. Sentry spans are telemetry, not an audit trail.

- **S L1:** Unstructured log lines for most tools; structured activity events only for autopilot workflows. Evidence: [packages/tm-core/src/modules/storage/adapters/activity-logger.ts:62](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/tm-core/src/modules/storage/adapters/activity-logger.ts#L62); [packages/tm-core/src/modules/workflow/managers/workflow-state-manager.ts:288](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/tm-core/src/modules/workflow/managers/workflow-state-manager.ts#L288) (verified)
  - *To reach the next level:* Write a structured record of every tool call with arguments and outcome.
- **C L1:** Only the autopilot workflow is recorded durably. Evidence: [packages/tm-core/src/modules/workflow/managers/workflow-state-manager.ts:288](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/tm-core/src/modules/workflow/managers/workflow-state-manager.ts#L288); [mcp-server/src/tools/tool-registry.js:59-104](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/mcp-server/src/tools/tool-registry.js#L59-L104) (verified)
  - *To reach the next level:* Record every tool call, including provider subprocess activity.
- **D L1:** Logging is on by default but persisted only for autopilot, in a user-writable location. Evidence: [packages/tm-core/src/modules/workflow/managers/workflow-state-manager.ts:42-44](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/tm-core/src/modules/workflow/managers/workflow-state-manager.ts#L42-L44) (verified)
  - *To reach the next level:* Persist records outside anything the agent's tools can edit.
- **B L1:** Records are best-effort; actions proceed if logging fails. Evidence: [packages/tm-core/src/modules/storage/adapters/activity-logger.ts:62](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/tm-core/src/modules/storage/adapters/activity-logger.ts#L62) (verified)
  - *To reach the next level:* Flush a durable record per action and surface failures.
- **Cap:** none

### C10 Limits & kill switch: 0.33 (high confidence)

Each model call has a per-role token limit and at most two retries, research file context is capped at 50KB, and the Grok CLI provider has a timeout that kills the process. There is no cost ceiling and no limit on how many model calls a bulk operation such as expand_all makes. Defaults can be raised from project configuration.

- **S L2:** Server-enforced per-call token limits, retry caps, a file-size cap and a timeout on the Grok CLI. Evidence: [scripts/modules/ai-services-unified.js:673](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/ai-services-unified.js#L673); [scripts/modules/ai-services-unified.js:245](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/ai-services-unified.js#L245); [scripts/modules/utils/contextGatherer.js:660](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/utils/contextGatherer.js#L660); [packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts:138-141](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/packages/ai-sdk-provider-grok-cli/src/grok-cli-language-model.ts#L138-L141) (verified)
  - *To reach the next level:* Cap every operation, including the number of model calls per request, and add a spend limit.
- **C L1:** Limits apply per model call; bulk tools and other provider CLIs have no overall bound. Evidence: [scripts/modules/ai-services-unified.js:673](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/ai-services-unified.js#L673); searched `rg -n -S -e 'max_?cost|costLimit|budget'` in `mcp-server/src apps/mcp/src src scripts/modules` → 0 hits (no spend or cost ceiling) (verified)
  - *To reach the next level:* Apply limits per tool request and to every provider subprocess.
- **D L1:** Defaults exist but are large (64,000 tokens) and come from project configuration. Evidence: [scripts/modules/config-manager.js:32](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/config-manager.js#L32); [scripts/modules/config-manager.js:136-144](https://github.com/eyaltoledano/claude-task-master/blob/c0c98d367c55296bfe69e65680625b6db437af02/scripts/modules/config-manager.js#L136-L144) (verified)
  - *To reach the next level:* Use tight defaults that workspace files can't raise.
- **B L1:** With no cost ceiling, a runaway bulk operation spends until every task is processed. Evidence: searched `rg -n -S -e 'max_?cost|costLimit|budget'` in `mcp-server/src apps/mcp/src src scripts/modules` → 0 hits (no spend or cost ceiling) (verified)
  - *To reach the next level:* Add a per-run spend ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: PRD files and web research results (mcp-server/src/tools/research.js:32) · [B] sensitive data/systems: Any readable local file and provider API keys (scripts/modules/utils/contextGatherer.js:646) · [C] state change / egress: Research provider and model endpoints, file writes and git commits (apps/mcp/src/tools/autopilot/commit.tool.ts:107) · Same default session? Yes

## Highest-impact improvements
1. Require a workspace-trust decision before project config or .env can change providers, endpoints or provider settings. (C6 S L0→L3, +0.225 before caps)
2. Make Sentry telemetry opt-in and stop recording prompts and responses. (C8 D L0→L2, +0.100 before caps)
3. Contain project root and file-path arguments to an approved project directory. (C3 S L2→L3, +0.075 before caps)
4. Scrub the environment passed to provider CLIs so each gets only its own credentials. (C7 B L0→L2, +0.100 before caps)
5. Add a dry-run preview for destructive task tools. (C2 S L2→L3, +0.075 before caps)

## Re-audit log
- C2 C: L3 → L2. Agent-CLI providers act inside a single tool call outside the host gate, and not every read-only hint was checked against the write paths.
- C3 B: L2 → L1. Tools aren't scoped to the workspace: path arguments accept any absolute path.

## Limitations
- Static source review at the pinned commit only; nothing was installed, built or run.
- Behaviour of the third-party provider SDKs (ai-sdk-provider-claude-code, codex-cli, gemini-cli) was not read; their subprocess environment and permission handling are inferred from their documented options.
- The CLI (task-master), including its loop, start and auto-update commands, and the VS Code extension were not scored; they are footnoted only.
- Hamster remote-storage mode (opt-in login) was not scored in depth.
