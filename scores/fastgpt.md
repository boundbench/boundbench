# Defense-in-depth score: FastGPT

**Repo:** https://github.com/labring/FastGPT · **Commit:** `c01b99ba549a44a7415ad2d660802b8211ace89f` · **Reviewed:** 2026-10-05
**What it is:** Self-hosted, multi-tenant platform for building LLM workflow and agent apps with knowledge bases, plugins, MCP tools and code nodes.
**Category:** Agent Frameworks
**Scored configuration:** Shipped Docker Compose deployment (deploy/version/main) with its defaults: workflow, tool-call and agent apps with designer-selected tools, code nodes in the code-sandbox service (internal-IP check on there), the OpenSandbox Agent Sandbox provider configured and enabled per app, and no internal-IP check for the app's own outbound tool requests.
**Agent surface (default):** code execution yes · filesystem write opt-in · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 3.9 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L0 | L0 | 0.30 | none | **0.30** | High |
| C2 | Approval gates | L1 | L0 | L0 | L0 | 0.07 | G1 | **0.07** (alt) | High |
| C3 | Tool & action scoping | L2 | L2 | L3 | L1 | 0.50 | none | **0.50** | High |
| C4 | Code-execution isolation | L3 | L3 | L2 | L2 | 0.65 | none | **0.65** | High |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L2 | L2 | L2 | L0 | 0.40 | none | **0.40** | High |
| C7 | Third-party extensions | L2 | L1 | L2 | L2 | 0.42 | none | **0.42** | Medium |
| C8 | Secrets & sensitive-data protection | L0 | L1 | L1 | L1 | 0.17 | none | **0.17** | High |
| C9 | Audit & traceability | L2 | L3 | L2 | L1 | 0.53 | none | **0.53** | High |
| C10 | Limits & kill switch | L2 | L3 | L3 | L1 | 0.57 | none | **0.57** | High |


FastGPT isolates code well: code nodes run in a separate sandbox service with chroot, a dropped user ID and a default-deny seccomp filter, and the main app never executes code itself. The dominant risk is everything around the tools: there is no approval step for any tool call, and an app that reads untrusted web pages, files or share-link messages can use its stored credentials to send data out or change external systems unattended. Private-network addresses are reachable from tools unless the operator turns on the internal-IP check, and the shipped deployment's default access configuration needs hardening before exposure.

## Critical gaps
- A hijacked app can leak knowledge-base data and take irreversible external actions through its HTTP, MCP and plugin tools with no person involved. (ASI01, LLM01; C5). Evidence: [packages/service/core/workflow/dispatch/tools/http468.ts:551-565](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/tools/http468.ts#L551-L565); [packages/service/support/outLink/runtime/service.ts:408](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/support/outLink/runtime/service.ts#L408)
- No tool call, including HTTP writes, MCP tools, plugins and the sandbox shell, requires human approval. (ASI09, ASI02; C2). Evidence: [packages/service/core/workflow/dispatch/ai/toolcall/toolProvider/createToolCallToolProvider.ts:115](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/ai/toolcall/toolProvider/createToolCallToolProvider.ts#L115)
- The default access configuration is not locked down, and the platform-level credentials reach every tenant's stored data. (ASI03; C1). Evidence: [packages/service/env.ts:38-46](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/env.ts#L38-L46); [packages/service/env.ts:246](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/env.ts#L246)

## Criterion details

### C1 Identity & least privilege: 0.30 (high confidence)

FastGPT is a multi-tenant platform: each app runs inside its team, and a published app is bound to the list of knowledge bases and tools declared in its version, which the runtime checks before loading anything. Tools then act with whatever credentials the app designer stored for them, and the same credentials serve every user of the app, including anonymous visitors of a share link, who run as the app owner. The quick start documents a fixed default administrator password, and the shipped deployment's default access configuration is not locked down, so least privilege depends on the operator hardening the install. The platform-level credentials reach every tenant's data.

- **S L2:** Runs are bound to the app's team and to the resources declared in its published version; undeclared apps or tools are refused. Evidence: [packages/service/core/workflow/utils/resource.ts:82-103](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/utils/resource.ts#L82-L103); [packages/service/support/outLink/runtime/service.ts:408](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/support/outLink/runtime/service.ts#L408) (verified)
  - *To reach the next level:* No per-capability credential scoping: read and write tools in an app share the credentials the designer stored.
- **C L2:** Built-in node types load apps and datasets through the snapshot check with a team filter, but tools, plugins and MCP servers use their own stored credentials and nothing is evaluated against the requesting user. Evidence: [packages/service/core/workflow/utils/resource.ts:315](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/utils/resource.ts#L315); [packages/service/support/outLink/runtime/service.ts:405-412](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/support/outLink/runtime/service.ts#L405-L412) (verified)
  - *To reach the next level:* Authorization is not evaluated against the requesting principal; share-link visitors act with the app owner's identity.
- **D L0:** Beyond the documented root login, the shipped deployment's platform-level keys are fixed values rather than generated per install, so the default identity setup is not locked down. Evidence: [README_en.md:48](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/README_en.md#L48); [packages/service/env.ts:38-46](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/env.ts#L38-L46) (verified)
  - *To reach the next level:* Generate platform keys and administrator credentials per install and require changing them before first use.
- **B L0:** The default access configuration is not locked down, and the platform-level credentials reach every tenant's stored data and files as well as each app's tool credentials. Evidence: [packages/service/env.ts:246](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/env.ts#L246); [packages/service/core/workflow/dispatch/tools/http468.ts:181](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/tools/http468.ts#L181) (verified)
  - *To reach the next level:* Platform and tool credentials are neither per-install secrets nor limited to one tenant.
- **Cap:** none

### C2 Approval gates: 0.07 (high confidence)

There is no approval step for tool calls. The tool-call and agent nodes execute whatever tool the model selects (HTTP requests, MCP tools, plugins, sub-apps, sandbox shell when enabled) without asking a person. Designers can place a user-choice or form step in a classic workflow, but it shows designer-written options rather than the exact call, and the tool-call and agent paths never pass through it. Consequential actions such as HTTP writes or messages sent through plugins cannot be undone.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** No approval mechanism exists for tool calls in the tool-call node or the agent loop. Evidence: searched `rg -n -S -i 'requireApproval|requiresApproval|toolApproval|approveTool|humanApproval|confirmToolCall'` in `packages/service packages/global projects/app/src` → 0 hits (no approval primitive anywhere in the server or app sources); [packages/service/core/workflow/dispatch/constants.ts:57](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/constants.ts#L57) (verified)
    - *To reach the next level:* No per-call human approval showing the exact tool and arguments.
  - **C L0:** The tool runner is called directly from the tool-call provider with no gate in between. Evidence: [packages/service/core/workflow/dispatch/ai/toolcall/toolProvider/createToolCallToolProvider.ts:115](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/ai/toolcall/toolProvider/createToolCallToolProvider.ts#L115) (verified)
    - *To reach the next level:* The most powerful paths (HTTP, MCP, plugins, sandbox shell) are not gated.
  - **D L0:** There is no approval to enable. Evidence: searched `rg -n -S -i 'requireApproval|requiresApproval|toolApproval|approveTool|humanApproval|confirmToolCall'` in `packages/service packages/global projects/app/src` → 0 hits (verified)
    - *To reach the next level:* No approval exists on by default.
  - **B L0:** Model-chosen HTTP tools can send POST/PUT/PATCH requests with model-filled bodies, which are irreversible external actions. Evidence: [packages/service/core/workflow/dispatch/tools/http468.ts:551-565](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/tools/http468.ts#L551-L565) (verified)
    - *To reach the next level:* No undo, preview or quantity bound on external actions.
- **designer-placed user-choice / form steps in classic workflows** (alt; raw 0.07, cap G1 → 0.07) ← counted
  - **S L1:** A designer can insert a user-choice step before a node; the person sees designer-written options, not the exact call. Evidence: [packages/service/core/workflow/dispatch/constants.ts:70](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/constants.ts#L70) (verified)
    - *To reach the next level:* The approver does not see the exact call and arguments.
  - **C L0:** Tool-call and agent nodes execute tools without passing through such a step. Evidence: [packages/service/core/workflow/dispatch/constants.ts:51-57](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/constants.ts#L51-L57) (verified)
    - *To reach the next level:* Model-selected tool calls bypass designer-placed steps.
  - **D L0:** Only present when a designer adds it to a workflow. Evidence: [packages/service/core/workflow/dispatch/constants.ts:70](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/constants.ts#L70) (verified)
    - *To reach the next level:* Not on by default.
  - **B L0:** Same irreversible external actions as the default path. Evidence: [packages/service/core/workflow/dispatch/tools/http468.ts:551-565](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/tools/http468.ts#L551-L565) (verified)
    - *To reach the next level:* No undo or preview for external actions.
- **Cap:** G1: Opt-in mechanism: off in the scored default configuration.

### C3 Tool & action scoping: 0.50 (high confidence)

Each app receives only the tools its designer selected, and the runtime refuses tools that are not declared in the published version. Outbound URLs from HTTP tools, MCP connections and file fetches go through a shared address check that always blocks loopback and cloud-metadata addresses and re-checks every redirect against a pinned DNS answer. Private-network addresses, including the deployment's own internal services, are only blocked when the operator turns on the internal-IP check, which is off by default. The HTTP tool itself stays general-purpose, with no host allowlists or quantity bounds.

- **S L2:** Typed tool parameters plus a URL guard with redirect re-checks and DNS pinning; private ranges are only checked when the internal-IP option is on, and the HTTP tool remains general. Evidence: [packages/global/common/system/network.ts:172-178](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/global/common/system/network.ts#L172-L178); [packages/service/common/api/axios.ts:63-66](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/common/api/axios.ts#L63-L66); [packages/service/core/workflow/dispatch/tools/http468.ts:551](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/tools/http468.ts#L551) (verified)
  - *To reach the next level:* General HTTP and shell tools are not replaced by narrow ones, and there are no per-tool host allowlists or numeric bounds.
- **C L2:** HTTP nodes, MCP connections and chat file URL fetches go through the guard; plugin tools validate their own arguments in the separate plugin service. Evidence: [packages/service/core/app/mcp.ts:130-133](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/app/mcp.ts#L130-L133); [packages/service/core/chat/fileContext.ts:488](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/chat/fileContext.ts#L488) (verified)
  - *To reach the next level:* No shared validation layer wraps plugin and MCP tool arguments.
- **D L3:** New apps have no tools; the designer selects each one and the run is restricted to the declared resource snapshot. Evidence: [packages/service/core/workflow/utils/resource.ts:95-101](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/utils/resource.ts#L95-L101); [packages/service/core/workflow/utils/resource.ts:299](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/utils/resource.ts#L299) (verified)
  - *To reach the next level:* Dynamically referenced apps are authorized by the runner's read permission rather than the declared snapshot, so per-task narrowing is not complete.
- **B L1:** Tools can reach any public host and, by default, private-network hosts as well. Evidence: [packages/service/env.ts:334](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/env.ts#L334) (verified)
  - *To reach the next level:* Private-network reach is on by default and no tool has quantity bounds.
- **Cap:** none

### C4 Code-execution isolation: 0.65 (high confidence)

The main application never executes code itself. Code nodes are sent to a separate code-sandbox service that runs each task in a fresh process inside a chroot, with a dropped user ID, no-new-privileges and a default-deny seccomp filter that blocks direct network access; outbound HTTP from code goes through a helper that blocks internal addresses and caps the number of requests. The optional Agent Sandbox, which gives the model a shell, runs in per-session containers created by OpenSandbox or Sealos with internal networks denied but public egress open by design. Turning off seccomp needs an explicit setting, and there is no fallback to running code on the host.

- **S L3:** Code tasks run in one-shot processes behind chroot, setresuid, no_new_privs and a default-deny seccomp allowlist. Evidence: [projects/code-sandbox/native/js-sandbox/fastgpt_js_sandbox.c:70](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/projects/code-sandbox/native/js-sandbox/fastgpt_js_sandbox.c#L70); [projects/code-sandbox/native/js-sandbox/fastgpt_js_sandbox.c:136-141](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/projects/code-sandbox/native/js-sandbox/fastgpt_js_sandbox.c#L136-L141) (verified)
  - *To reach the next level:* Not kernel-separated isolation (microVM, gVisor or WASM); the Agent Sandbox shell uses ordinary containers.
- **C L3:** No code execution in the main app; code nodes error out if the sandbox URL is missing; the model shell only runs inside the provider container. Evidence: searched `rg -n -S 'child_process|new Function\(|\beval\(|runInNewContext'` in `packages/service packages/global projects/app/src` → 0 hits (no in-process or host execution in the main application); [packages/service/core/workflow/dispatch/tools/codeSandbox.ts:33](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/tools/codeSandbox.ts#L33); [packages/service/core/ai/sandbox/application/toolCall/shell.tool.ts:21-23](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/ai/sandbox/application/toolCall/shell.tool.ts#L21-L23) (verified)
  - *To reach the next level:* An operator escape hatch disables seccomp, and the separate plugin service runtime was not verified here.
- **D L2:** Seccomp is on unless the operator sets an explicit flag and the model cannot request unsandboxed execution, but access to the sandbox services' controls is not locked down in the default deployment. Evidence: [projects/code-sandbox/src/env.ts:55](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/projects/code-sandbox/src/env.ts#L55); [packages/service/env.ts:69-75](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/env.ts#L69-L75) (verified)
  - *To reach the next level:* Sandbox service access should be protected by per-install secrets before seccomp-off and similar changes count as operator-only.
- **B L2:** Code tasks are ephemeral with a scrubbed environment and no direct sockets, but public egress through the helper is not allowlisted; the opt-in Agent Sandbox keeps a per-session volume with open public egress. Evidence: [projects/code-sandbox/src/pool/base-process-pool.ts:154-157](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/projects/code-sandbox/src/pool/base-process-pool.ts#L154-L157); [projects/code-sandbox/src/env.ts:65-66](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/projects/code-sandbox/src/env.ts#L65-L66); [packages/service/core/ai/sandbox/infrastructure/provider/runtimeProfile/opensandbox.ts:48](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/ai/sandbox/infrastructure/provider/runtimeProfile/opensandbox.ts#L48) (verified)
  - *To reach the next level:* Egress is not off or allowlisted.
- **Cap:** none

### C5 Untrusted input blast radius: 0.25 (high confidence)

Nothing structural limits what a hijacked app can do. Knowledge-base quotes are wrapped in delimiter tags in the default prompt template and tool results arrive in the tool role, but web pages, uploaded files, MCP and HTTP results, and messages from public share-link visitors are not treated differently from instructions. In normal use an app reads untrusted content, holds team knowledge and tool credentials, and can send data out or change external systems through HTTP, MCP and plugin tools in the same run with no person involved.

- **S L1:** Only delimiters around knowledge-base quotes; no detection, taint tracking or approval tied to untrusted content. Evidence: [packages/global/core/ai/prompt/AIChat.ts:31](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/global/core/ai/prompt/AIChat.ts#L31); searched `rg -n -S -i 'prompt.?injection|jailbreak|untrusted'` in `packages/service/core/ai packages/service/core/workflow` → 0 hits (verified)
  - *To reach the next level:* No approval or capability restriction once untrusted content has been read.
- **C L1:** Knowledge-base quotes are delimited; files, web pages, tool results and other users' messages are not distinguished. Evidence: [packages/service/core/ai/llm/agentLoop/provider/piAgent/run.ts:183](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/ai/llm/agentLoop/provider/piAgent/run.ts#L183) (verified)
  - *To reach the next level:* Tool results, files and third-party messages are not treated as data.
- **D L2:** The delimiting quote template is on by default, but app designers can replace it without any warning. Evidence: [packages/global/core/ai/prompt/AIChat.ts:12](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/global/core/ai/prompt/AIChat.ts#L12); [packages/global/core/workflow/node/constant.ts:27](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/global/core/workflow/node/constant.ts#L27) (verified)
  - *To reach the next level:* Replacing the template is not warned.
- **B L0:** Untrusted input, team knowledge and credentials, and outbound HTTP writes combine in one unattended run. Evidence: [packages/service/core/workflow/dispatch/tools/http468.ts:551-565](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/tools/http468.ts#L551-L565); [packages/service/core/chat/fileContext.ts:488](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/chat/fileContext.ts#L488); [packages/service/support/outLink/runtime/service.ts:408](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/support/outLink/runtime/service.ts#L408) (verified)
  - *To reach the next level:* No approval for exfiltration or irreversible actions after untrusted content is read.
- **Cap:** C5-WORSTCASE: Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity: 0.40 (high confidence)

The model has no long-term memory tool: knowledge bases are filled by people (uploads, imports, website and API sync), skills are saved and deployed as versions through an explicit action, and agent memory and variables live inside a single chat. Knowledge bases are team-scoped and chats are per user, enforced in queries. Imported knowledge-base content, including synced web pages, is not validated and has no expiry, and once in a knowledge base it is retrieved for every user of the app and can steer tool use.

- **S L2:** No model-writable long-term memory; skills are published through an explicit deploy action; retrieved quotes are presented inside delimiters. Evidence: searched `rg -n -S -i 'longTermMemory|saveMemory|userMemory'` in `packages/service` → 0 hits; [packages/service/core/ai/sandbox/application/skillEdit/deploy.ts:2](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/ai/sandbox/application/skillEdit/deploy.ts#L2); [packages/global/core/ai/prompt/AIChat.ts:31](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/global/core/ai/prompt/AIChat.ts#L31) (verified)
  - *To reach the next level:* Imported and synced knowledge-base content has no validation, provenance gate or expiry.
- **C L2:** Chat-scoped memory and skills are controlled; knowledge-base imports and syncs are not. Evidence: [packages/service/core/workflow/dispatch/constants.ts:53](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/constants.ts#L53) (verified)
  - *To reach the next level:* Retrieval stores accept content without a gate.
- **D L2:** Datasets and apps are filtered by team, chats by user, in queries. Evidence: [packages/service/core/workflow/utils/resource.ts:259](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/utils/resource.ts#L259); [packages/service/core/chat/chatSchema.ts:76](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/chat/chatSchema.ts#L76) (verified)
  - *To reach the next level:* Root test and debug runs skip the team filter, so isolation is not fixed for every caller.
- **B L0:** Poisoned knowledge-base content persists for every user of the app and can trigger tool use without approval. Evidence: [packages/service/core/workflow/dispatch/constants.ts:53](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/constants.ts#L53); [packages/service/core/workflow/dispatch/ai/toolcall/toolProvider/createToolCallToolProvider.ts:115](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/ai/toolcall/toolProvider/createToolCallToolProvider.ts#L115) (verified)
  - *To reach the next level:* No human review or rollback before imported content influences tool-using runs.
- **Cap:** none

### C7 Third-party extensions: 0.42 (medium confidence)

Extensions come as system plugins served by a separate plugin service, team-installed plugins, and remote MCP servers. Plugin installs record the version, an etag and the permissions confirmed at install time, and team plugin installation is off unless the operator enables it; package verification itself happens in the plugin service, which is not part of this repository. MCP servers are remote URLs whose code and tool definitions are not pinned. No extension runs inside the main application process.

- **S L2:** Plugin installs are recorded with version, etag and confirmed permissions; integrity verification, if any, lives in the separate plugin service. Evidence: [projects/app/src/pages/api/core/plugin/team/pkg/installWithUrl.ts:71-79](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/projects/app/src/pages/api/core/plugin/team/pkg/installWithUrl.ts#L71-L79); [packages/service/core/app/mcp.ts:401](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/app/mcp.ts#L401) (inferred)
  - *To reach the next level:* No hash or signature verification in this repository and no re-approval when an extension changes.
- **C L1:** Plugins are tracked by version; MCP servers are used as whatever the remote URL serves. Evidence: [packages/service/core/app/mcp.ts:401-417](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/app/mcp.ts#L401-L417) (verified)
  - *To reach the next level:* MCP servers and their tool definitions are not pinned or verified.
- **D L2:** Team plugin installation is refused unless the operator enables it, and installing requires team-manage permission. Evidence: [packages/service/core/plugin/teamPluginPolicy.ts:43-46](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/plugin/teamPluginPolicy.ts#L43-L46); [projects/app/src/pages/api/core/plugin/team/pkg/installWithUrl.ts:41-45](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/projects/app/src/pages/api/core/plugin/team/pkg/installWithUrl.ts#L41-L45) (verified)
  - *To reach the next level:* Which bundled system tools are enabled by default is decided by the separate plugin service and was not verified.
- **B L2:** Plugins run in a separate service and MCP servers are remote; nothing is launched in the main process. Evidence: searched `rg -n -S 'StdioClientTransport'` in `packages/service projects/app/src` → 0 hits (no locally launched MCP servers); [packages/service/core/app/mcp.ts:401](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/app/mcp.ts#L401) (inferred)
  - *To reach the next level:* Plugins share one plugin service instead of a per-extension sandbox with scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection: 0.17 (high confidence)

Credentials that designers attach to HTTP and MCP tools are encrypted at rest with AES-256-GCM, decrypted only when the request is sent, and left out of the recorded node responses, so they never reach the model. However, the shipped default configuration carries fixed secret values, including the key that protects those stored credentials, so the at-rest protection depends on the operator replacing them. Model provider keys and inter-service tokens are plain configuration values, logs have no redaction, and OpenTelemetry export is off unless configured.

- **S L0:** Tool secrets are encrypted at rest and injected only at request time, but the shipped default configuration carries fixed secret values, including the at-rest encryption key; logs are not redacted. Evidence: [packages/service/env.ts:42-43](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/env.ts#L42-L43); [packages/service/core/workflow/dispatch/tools/http468.ts:181](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/tools/http468.ts#L181); searched `rg -n -S -i 'redact|desensitiz'` in `packages/service/common/logger` → 0 hits (verified)
  - *To reach the next level:* No secret material in shipped defaults; keys generated per install or held in a secret manager.
- **C L1:** Model context and saved node responses exclude tool header secrets; logs and errors are not covered. Evidence: [packages/service/core/workflow/dispatch/tools/http468.ts:301](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/tools/http468.ts#L301) (verified)
  - *To reach the next level:* Logs, error messages and the at-rest encryption key are not protected.
- **D L1:** OpenTelemetry logs and traces are off by default; console logs default to debug level. Evidence: [packages/service/env.ts:297-308](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/env.ts#L297-L308) (verified)
  - *To reach the next level:* Shipped defaults include fixed secret values and there is no always-on redaction.
- **B L1:** Long-lived tool and model keys, plus documented default inter-service tokens. Evidence: [packages/service/env.ts:51-55](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/env.ts#L51-L55) (verified)
  - *To reach the next level:* Keys are long-lived and not scoped per task or rotated.
- **Cap:** none

### C9 Audit & traceability: 0.53 (high confidence)

Every node run, including tool calls, MCP and plugin tools and child apps, is stored as a structured row tied to the chat, which records the team member or share-link user. Team configuration changes go to a separate team audit log. Records live in the application database, are deleted together with chat history, and are written in batches; write failures are logged and the run continues.

- **S L2:** Structured per-node records attached to chats that carry the requesting member or share-link user. Evidence: [packages/service/core/chat/nodeResponseStorage.ts:443](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/chat/nodeResponseStorage.ts#L443); [packages/service/core/chat/chatSchema.ts:26](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/chat/chatSchema.ts#L26) (verified)
  - *To reach the next level:* No approver or delegation-chain attribution and no tamper-evident storage.
- **C L3:** All node runs, including tools, extensions and child apps, are recorded, and configuration changes are audited. Evidence: [packages/service/support/user/audit/util.ts:108-114](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/support/user/audit/util.ts#L108-L114); [packages/service/core/chat/nodeResponseStorage.ts:443](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/chat/nodeResponseStorage.ts#L443) (verified)
  - *To reach the next level:* Credential use is not recorded.
- **D L2:** On by default and written by the platform, but stored in the app database and removed with chat deletion. Evidence: [packages/service/core/chat/delete.ts:82](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/chat/delete.ts#L82) (verified)
  - *To reach the next level:* Records are not written by a component separate from the application's own data store.
- **B L1:** Rows are flushed in batches of five and write failures do not stop the run; audit write failures are only logged. Evidence: [packages/service/core/chat/nodeResponseStorage.ts:428-431](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/chat/nodeResponseStorage.ts#L428-L431); [packages/service/support/user/audit/util.ts:116](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/support/user/audit/util.ts#L116) (verified)
  - *To reach the next level:* Records are not durable per action.
- **Cap:** none

### C10 Limits & kill switch: 0.57 (high confidence)

Runs are bounded by a node-run budget (500 by default) that child apps draw from, by round caps in the agent loop (100) and tool-call node (50), by loop and parallelism caps, and by per-request timeouts. A stop request is polled every 100 ms and checked between nodes and model calls. There is no token or cost cap in the community setup (team AI-point budgets only apply with subscription plans), in-flight tool calls finish after a stop, and Agent Sandbox containers stay up until the inactivity timeout (60 minutes by default).

- **S L2:** Iteration caps plus per-request timeouts and a cooperative stop; no cost cap without subscription plans. Evidence: [packages/service/env.ts:406](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/env.ts#L406); [packages/service/core/workflow/dispatch/ai/toolcall/toolCall.ts:166](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/ai/toolcall/toolCall.ts#L166); [packages/service/support/permission/teamLimit.ts:15](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/support/permission/teamLimit.ts#L15) (verified)
  - *To reach the next level:* No token or cost cap per run in the default community deployment.
- **C L3:** Child apps deduct from the parent's run budget and parallel branches are capped. Evidence: [packages/service/core/workflow/dispatch/index.ts:1201-1202](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/index.ts#L1201-L1202); [packages/service/env.ts:415](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/env.ts#L415) (verified)
  - *To reach the next level:* No explicit cap on sub-app nesting depth.
- **D L3:** Sensible defaults set by the operator; the model cannot raise them. Evidence: [packages/service/env.ts:406](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/env.ts#L406) (verified)
  - *To reach the next level:* No hard ceiling that configuration cannot exceed.
- **B L1:** No spend ceiling; a stop ends the loop between nodes while in-flight calls complete and sandbox containers stay up until the inactivity suspend. Evidence: [packages/service/core/workflow/dispatch/index.ts:250-270](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/index.ts#L250-L270); [packages/service/core/workflow/dispatch/index.ts:1323](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/core/workflow/dispatch/index.ts#L1323); [packages/service/env.ts:126](https://github.com/labring/FastGPT/blob/c01b99ba549a44a7415ad2d660802b8211ace89f/packages/service/env.ts#L126) (verified)
  - *To reach the next level:* No spend ceiling, and stopping does not cancel in-flight tool calls or sandbox work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Files and URLs fetched for chats (packages/service/core/chat/fileContext.ts:488), web/API knowledge-base content, tool results, and share-link visitors (packages/service/support/outLink/runtime/service.ts:408) · [B] sensitive data/systems: Team knowledge bases and tool credentials decrypted at request time (packages/service/core/workflow/dispatch/tools/http468.ts:181) · [C] state change / egress: HTTP tool writes to any public host (packages/service/core/workflow/dispatch/tools/http468.ts:551-565), MCP and plugin tools · Same default session? Yes

## Highest-impact improvements
1. Turn the internal-IP check on by default so tools cannot reach private-network services. (C3 B L1→L2, +0.050 before caps; Playbook 3)
2. Add a per-call approval step, showing the exact tool and arguments, for tools the designer marks as consequential. (C2 S L0→L3, +0.225 before caps; Playbook 5)
3. Once untrusted content enters a run, require approval for egress and write tools. (C5 S L1→L2, +0.075 before caps; Playbook 1)
4. Add a per-run token or cost cap that applies without subscription plans. (C10 S L2→L3, +0.075 before caps; Playbook 3 step 3)
5. Generate platform keys and administrator credentials per install instead of shipping fixed values. (C1 D L0→L2, +0.100 before caps; Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The fastgpt-plugin service, OpenSandbox server and Sealos Devbox runtimes are separate projects and were not examined; plugin verification and confinement are rated from what this repository shows (marked inferred).
- The commercial (pro) features and subscription-plan budgets were not scored; the community configuration was.
