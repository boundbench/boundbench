# Defense-in-Depth Score: Skyvern

**Repo:** https://github.com/Skyvern-AI/skyvern · **Commit:** `19bf399835a029b70a9fcec6dcea347a1c40c033` · **Reviewed:** 2026-10-03
**What it is:** Automates browser-based workflows with LLMs and vision
**Category:** AI Assistants
**Scored configuration:** pip install "skyvern[all]" + `skyvern quickstart` (local API + UI, SQLite, settings defaults), tasks on the default skyvern-1.0 engine and workflows in agent mode.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents yes · external communication yes

## Score: 3.4 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |
| C2 | Approval gates | L1 | L0 | L0 | L0 | 0.07 | G1 | **0.07** (alt) | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C4 | Code-execution isolation | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C7 | Third-party extensions | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L0 | L1 | 0.35 | — | **0.35** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |


Skyvern drives a real browser through arbitrary websites using stored logins, and as shipped nothing stands between a prompt-injected page and the agent's actions: there is no human approval, navigation can go to any public site, and credential release on the default engine is not a complete boundary. Workflow code blocks are on by default and run in the server process behind only an AST filter. The codebase contains a sophisticated origin firewall, egress monitor and credential site-binding, but in this open-source build the firewall cannot be enrolled and the site-binding does not cover every path. Secrets are kept out of the model's context via placeholders, which is a real strength.

## Critical gaps
- Default runs combine untrusted web content, stored credentials and unrestricted navigation/form submission with no human gate (C5-WORSTCASE). (ASI01, LLM01, T6; C5) — [skyvern/webeye/actions/handler.py:11351-11354](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/webeye/actions/handler.py#L11351-L11354)
- Code blocks are enabled by default and exec() Python in the API server process behind only an AST denylist; the OSS build has no isolated runner. (ASI05, T11, LLM05; C4) — [skyvern/forge/sdk/workflow/models/block.py:6834-6849](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/models/block.py#L6834-L6849); [skyvern/config.py:865](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/config.py#L865); [skyvern/forge/agent_functions.py:1296-1305](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/agent_functions.py#L1296-L1305)

## Criterion details

### C1 Identity & least privilege — 0.38 (high)

Skyvern acts on websites with whatever site passwords, TOTP secrets and password-manager credentials a workflow author binds as parameters, plus the operator's LLM API keys in the server's environment. A run only receives the credentials its workflow declares, and the API requires an org API key, but per-site credential release on the default task engine is not a complete boundary. A hijacked run can use the bound credentials across as many services as the workflow touches.

- **S L2:** Credentials are scoped per workflow run to the parameters the author declared (and OTP use is further restricted to a block's linked credential keys), but each credential is static for the run with no per-action or per-site authority. — [skyvern/forge/agent.py:900-921](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/agent.py#L900-L921) (verified)
  - *To reach the next level:* No short-lived or downscoped credentials, and per-site release is not a complete boundary.
- **C L1:** The CredentialReleaseGuard site policy does not cover every credential-release path. — [skyvern/forge/sdk/credential_site_policy.py:50-55](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/credential_site_policy.py#L50-L55) (verified)
  - *To reach the next level:* Not every tool path goes through the same authorization layer.
- **D L2:** The API refuses requests without an API key and the local vault is only populated by explicit credential creation, but the server binds 0.0.0.0 on non-Windows hosts and LLM keys live in the process environment. — [skyvern/forge/sdk/services/org_auth_service.py:170-174](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/services/org_auth_service.py#L170-L174); [skyvern/cli/run_commands.py:60-62](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/cli/run_commands.py#L60-L62) (verified)
  - *To reach the next level:* Default does not start from a read-only or minimal authority; credential use requires no explicit elevation per run.
- **B L1:** A hijacked run can log in and act with every website credential bound to the workflow (and Bitwarden/1Password service-account secrets if configured), i.e. write access across multiple external systems. — [skyvern/forge/sdk/workflow/context_manager.py:928-932](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/context_manager.py#L928-L932) (verified)
  - *To reach the next level:* Not confined to one system or tenant; credentials are long-lived site passwords.
- **Cap:** none

### C2 Approval gates — 0.07 (high)

There is no human approval step before the agent clicks, types, submits forms, uploads files, or navigates; actions proposed by the model are executed directly. The only human gate is an optional Human Interaction workflow block that pauses a workflow for a single approve/reject decision on an author-written instruction, not on the specific action. The code labelled 'effect approval' is a machine-to-machine binding for a cloud firewall that is not enrolled in this build. Consequential web actions (purchases, submissions, account changes) are irreversible and unattended by default.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** No per-action human approval exists in the default agent loop; the only 'approval' symbols belong to the unenrolled machine firewall. — searched `rg -n -i 'requires_approval|human_approval|approval_required'` in `skyvern` → 6 hits (All six hits are machine-authorization enums in the browser-effect firewall and the extension broker; none gates an agent action on a human.); [skyvern/forge/agent.py:5118-5121](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/agent.py#L5118-L5121) (verified)
    - *To reach the next level:* No human approval of consequential actions at all.
  - **C L0:** Every action type, including form submission, file upload and navigation, executes without crossing a gate. — [skyvern/webeye/actions/handler.py:11585-11589](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/webeye/actions/handler.py#L11585-L11589) (verified)
    - *To reach the next level:* No gate exists to cover any path.
  - **D L0:** No default-on approval; the HumanInteractionBlock must be placed in a workflow by its author. — [skyvern/forge/sdk/workflow/models/block.py:14648-14659](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/models/block.py#L14648-L14659) (verified)
    - *To reach the next level:* Approval is not on by default.
  - **B L0:** Wrongly taken actions are real web form submissions, payments, registrations and emails with no rollback or preview. — [skyvern/forge/sdk/workflow/models/block.py:12459](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/models/block.py#L12459); [skyvern/webeye/actions/handler.py:11351-11354](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/webeye/actions/handler.py#L11351-L11354) (verified)
    - *To reach the next level:* No checkpoints, dry-runs, or quantity bounds on external actions.
- **opt-in Human Interaction block (workflow-level go/no-go)** (alt; raw 0.07, cap G1 → 0.07) ← counted
  - **S L1:** A blanket approve/reject pause with author-written instructions; the approver does not see the exact upcoming actions. — [skyvern/forge/sdk/workflow/models/block.py:14648-14670](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/models/block.py#L14648-L14670) (verified)
    - *To reach the next level:* Not per-call and does not show exact arguments.
  - **C L0:** Covers only the point in the workflow where it is placed; all agent actions elsewhere bypass it. — [skyvern/forge/sdk/workflow/models/block.py:14648-14659](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/models/block.py#L14648-L14659) (verified)
    - *To reach the next level:* Most powerful action paths are not gated.
  - **D L0:** Opt-in per workflow. — [skyvern/forge/sdk/workflow/models/block.py:14648-14659](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/models/block.py#L14648-L14659) (verified)
    - *To reach the next level:* Not on by default.
  - **B L0:** Same irreversible web actions once approved or outside the block. — [skyvern/webeye/actions/handler.py:11351-11354](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/webeye/actions/handler.py#L11351-L11354) (verified)
    - *To reach the next level:* No rollback or bounds.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C3 Tool & action scoping — 0.45 (high)

The agent's actions are browser UI primitives (click an element, type text, select, upload, navigate) rather than shell or SQL, and several have real argument checks: navigation blocks internal and private addresses and re-checks every redirect, downloads go through an SSRF-guarded connector, and an upload URL must appear verbatim in the user's own goal or payload. But navigation accepts any public URL and text input accepts any value, so the effective reach is 'any website with any input'. There is no per-task action allowlist or read-only default.

- **S L2:** URL validation blocks internal hosts with DNS resolution and redirect re-validation, and upload sources are restricted to user-provided URLs, but navigation is an open public-URL tool and input text is unbounded. — [skyvern/webeye/actions/handler.py:11351-11354](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/webeye/actions/handler.py#L11351-L11354); [skyvern/webeye/navigation.py:82-96](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/webeye/navigation.py#L82-L96); [skyvern/webeye/actions/handler.py:10180-10204](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/webeye/actions/handler.py#L10180-L10204); [skyvern/utils/url_validators.py:239-249](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/utils/url_validators.py#L239-L249) (verified)
  - *To reach the next level:* No destination allowlist per task; general navigate/type tools are not replaced by narrow ones.
- **C L2:** Navigation, upload and download validate arguments; clicks on in-page links and text input carry no destination or value checks, and the browser egress firewall is unenrolled. — [skyvern/forge/sdk/api/files.py:435](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/api/files.py#L435); [skyvern/webeye/browser_factory.py:1449-1456](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/webeye/browser_factory.py#L1449-L1456) (verified)
  - *To reach the next level:* Validation is per-handler, not a central policy layer covering every action.
- **D L2:** Action groups follow the task type: a task with no navigation goal is an extraction task that only ever receives an extract action, but the default navigation task gets the full write/navigate/upload vocabulary. — [skyvern/forge/agent.py:4667](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/agent.py#L4667); [skyvern/forge/agent.py:5481-5489](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/agent.py#L5481-L5489); [skyvern/webeye/actions/handler.py:11565-11582](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/webeye/actions/handler.py#L11565-L11582) (verified)
  - *To reach the next level:* The default task type is not read-only and there is no per-task action allowlist.
- **B L1:** A misused action set can operate on any public website with any stored login, limited only by the internal-address block and step cap. — [skyvern/config.py:309](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/config.py#L309); [skyvern/config.py:176](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/config.py#L176) (verified)
  - *To reach the next level:* Not scoped to a project or set of sites.
- **Cap:** none

### C4 Code-execution isolation — 0.25 (high)

Workflow code blocks are enabled by default and run Python in the Skyvern server process via exec(), protected only by an AST denylist and a restricted builtins dict. The project's own comment says the OSS build has no secure runner (that is a cloud feature). Code reaching this path is not only operator-written: the 2.0 task planner asks the LLM to write 'compute' code and runs it through the same block, and opt-in cached scripts are imported with the genuine builtins. An escape from the filter lands in a process holding LLM API keys, the vault key, the database and unrestricted network.

- **S L1:** In-process exec of Python filtered by an AST denylist of attributes, imports and builtins; filtering a full-capability runtime, not an isolation boundary. — [skyvern/forge/sdk/workflow/models/block.py:6834-6849](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/models/block.py#L6834-L6849); [skyvern/forge/sdk/workflow/code_block_safety.py:22-23](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/code_block_safety.py#L22-L23); [skyvern/forge/agent_functions.py:1296-1305](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/agent_functions.py#L1296-L1305) (verified)
  - *To reach the next level:* No OS-level separation (container, low-privilege user) for code blocks in the OSS build.
- **C L1:** Code blocks and LLM-written task-v2 compute code go through the AST filter, but cached scripts (code mode) are loaded with importlib under the genuine builtins and OSS policy always allows in-process script execution. — [skyvern/services/task_v2_service.py:2045-2048](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/services/task_v2_service.py#L2045-L2048); [skyvern/services/script_service.py:3491-3495](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/services/script_service.py#L3491-L3495); [skyvern/forge/agent_functions.py:1361](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/agent_functions.py#L1361); [skyvern/forge/sdk/workflow/code_block_safety.py:749-751](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/code_block_safety.py#L749-L751) (verified)
  - *To reach the next level:* Not every execution path uses the same filter; the script path executes with full builtins.
- **D L2:** The filter is always applied to code blocks and cannot be switched off, but code blocks themselves are enabled by default (CODE_BLOCK_MODE=enabled) and the cloud runner selection hard-returns False in OSS. — [skyvern/config.py:865](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/config.py#L865); [skyvern/forge/agent_functions.py:1296-1305](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/agent_functions.py#L1296-L1305) (verified)
  - *To reach the next level:* No isolation boundary on by default to tamper-protect.
- **B L0:** An escape executes inside the API server process, which holds LLM provider keys, the credential vault and its key file, the database connection and unrestricted network. — [skyvern/forge/sdk/workflow/models/block.py:6834-6849](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/models/block.py#L6834-L6849); [skyvern/forge/sdk/services/credential/skyvern_credential_vault_service.py:158-165](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/services/credential/skyvern_credential_vault_service.py#L158-L165) (verified)
  - *To reach the next level:* Code runs with host-equivalent authority and credentials in the environment.
- **Cap:** none

### C5 Untrusted input blast radius — 0.25 (high)

Skyvern's core job is reading arbitrary web pages and acting on them, so every run combines untrusted input, stored credentials and the ability to submit forms and navigate anywhere. The defence is prompt-level: about a third of the prompt templates fence page content as untrusted data and neutralize delimiter look-alikes, but the 2.0 planner and many helper prompts do not. Nothing in the default build cuts a Rule-of-Two leg: an injected page can steer the agent to an attacker URL carrying data in the query string, with no human involved, and credential release is not a complete boundary. A detailed origin firewall exists in the code but cannot be enrolled in this build.

- **S L1:** Spotlighting only: an `untrusted` Jinja filter escapes code fences and neutralizes sentinels around page data plus a 'SECURITY BOUNDARY' instruction in the prompt. — [skyvern/forge/sdk/prompting.py:36-42](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/prompting.py#L36-L42); [skyvern/forge/prompts/skyvern/extract-action.j2:1](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/prompts/skyvern/extract-action.j2#L1) (verified)
  - *To reach the next level:* No structural restriction (approval or disabled egress) once untrusted content is read.
- **C L1:** 29 of 95 prompt templates apply the untrusted fencing; task-v2 planning, completion checks and many helper prompts ingest page-derived text without it. — searched `rg -l -i untrusted` in `skyvern/forge/prompts/skyvern` → 29 hits (Files (not lines) that mention untrusted; the directory holds 95 .j2 templates, so 66 do not, including task_v2_generate_task_block.j2 and task_v2_check_completion.j2.); [skyvern/forge/prompts/skyvern/task_v2_generate_task_block.j2:1](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/prompts/skyvern/task_v2_generate_task_block.j2#L1) (verified)
  - *To reach the next level:* Most untrusted sources (planner prompts, file text, extracted data passed between blocks) are not fenced.
- **D L2:** The fencing is baked into the templates and on by default; it is not configurable. — [skyvern/forge/sdk/prompting.py:36-42](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/prompting.py#L36-L42) (verified)
  - *To reach the next level:* No structural control exists to be on by default.
- **B L0:** A hijacked default run can exfiltrate via goto_url to any public host and submit irreversible forms, all unattended; credential release is not a complete boundary. — [skyvern/webeye/actions/handler.py:11351-11354](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/webeye/actions/handler.py#L11351-L11354); searched `rg -n replace_browser_action_policy` in `skyvern` → 1 hits (Only the definition in workflow/service.py; no OSS route or CLI calls the policy writer, so the browser action firewall cannot be enrolled in the shipped build.) (verified)
  - *To reach the next level:* No egress restriction or approval on the default path.
- **Cap:** C5-WORSTCASE — Default configuration lets a hijacked agent both leak data/credentials and take irreversible web actions with no human.

### C6 Memory, context & configuration integrity — 0.38 (high)

Skyvern has no model-writable long-term memory or vector store, and starts each default run in a fresh temporary browser profile. Persistence is opt-in: saved browser profiles and persistent sessions carry cookies and site state between runs, and 'code mode' caches LLM-generated scripts that later runs execute without a fresh model decision. These stores are scoped per organization in queries, but nothing validates or reviews what a poisoned session or cached script carries forward. Settings read a .env from the server's working directory, which is operator configuration rather than an untrusted workspace.

- **S L1:** No model-controlled memory; when enabled, persisted state (saved browser profiles, cached scripts generated from runs) is written and reused without validation, review, or provenance. — searched `rg -n -i 'save_memory|long_term_memory|vector_store|pgvector|chromadb'` in `skyvern` → 0 hits (No memory or vector-store subsystem.); [skyvern/webeye/browser_factory.py:1077-1078](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/webeye/browser_factory.py#L1077-L1078) (verified)
  - *To reach the next level:* No review gate, validation, or expiry on cached scripts and saved browser state.
- **C L2:** Default runs use fresh profiles and agent mode; the opt-in script cache and profile paths have no comparable control. — [skyvern/forge/sdk/workflow/models/workflow.py:250-257](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/models/workflow.py#L250-L257) (verified)
  - *To reach the next level:* Cached scripts and saved profiles are not covered by any validation.
- **D L2:** Stores are keyed by organization_id in queries; the OSS quickstart is single-org. — [skyvern/forge/sdk/workflow/service.py:17792-17795](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/service.py#L17792-L17795) (verified)
  - *To reach the next level:* No per-tenant storage separation or default retention limits.
- **B L1:** When code mode is on, a script generated from a poisoned run persists and drives browser actions in later runs; a saved profile carries poisoned site state across sessions. — [skyvern/services/script_service.py:3491-3495](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/services/script_service.py#L3491-L3495) (verified)
  - *To reach the next level:* Poisoned persisted state can trigger actions, not just text.
- **Cap:** none

### C7 Third-party extensions — 0.30 (high)

Skyvern does not load plugins, MCP servers, or model files at runtime, and the model cannot install packages. The one third-party extension point is an operator setting (EXTENSIONS) that loads unpacked Chromium extensions from a local directory into every browser it launches; the list is empty by default. Those extensions are not pinned or hash-checked and run inside the same browser where credentials are typed, but they cannot be added by the model or by page content.

- **S L1:** Operator-named extension directories are loaded as-is with no version pin or integrity check. — [skyvern/webeye/browser_factory.py:716-739](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/webeye/browser_factory.py#L716-L739); searched `rg -n 'trust_remote_code|pickle\.load|torch\.load|stdio_client|StdioServerParameters|load_plugin'` in `skyvern` → 0 hits (No model loading, MCP client, or plugin loader in the server.) (verified)
  - *To reach the next level:* No pinning or hash verification of extension contents.
- **C L1:** The only extension type (browser extensions) is unverified; no other type exists. — [skyvern/config.py:436-437](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/config.py#L436-L437) (verified)
  - *To reach the next level:* No verification on the one extension type.
- **D L2:** Nothing third-party is enabled by default and only operator configuration can add extensions. — [skyvern/config.py:436-437](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/config.py#L436-L437) (verified)
  - *To reach the next level:* Mechanism strength caps this; adding an extension does not show its permissions.
- **B L1:** An extension runs in Chromium's extension process without the server environment, but shares the agent's browser and can read every page, including credentials Skyvern types. — [skyvern/webeye/browser_factory.py:735-738](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/webeye/browser_factory.py#L735-L738) (verified)
  - *To reach the next level:* Not sandboxed per extension with its own scoped access.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.35 (high)

Secrets are handled better than average: stored credentials appear to the model only as random placeholders that are swapped for real values at execution time, logs pass through field-name and bearer-token redaction, and the local vault is Fernet-encrypted. Weak spots: the vault's key file sits next to it, general database encryption is off by default, and telemetry to PostHog is on by default and sends each task's URL (which can carry tokens or personal data). The credentials themselves are long-lived site passwords.

- **S L2:** Opaque placeholders keep secrets out of model context and log processors redact sensitive fields, but the vault key is a plaintext file beside the vault and ENABLE_ENCRYPTION defaults to False. — [skyvern/forge/sdk/workflow/context_manager.py:115](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/context_manager.py#L115); [skyvern/forge/sdk/services/credential/skyvern_credential_vault_service.py:153-165](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/services/credential/skyvern_credential_vault_service.py#L153-L165); [skyvern/config.py:938](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/config.py#L938) (verified)
  - *To reach the next level:* No OS keychain or external secret manager by default; key material is co-located with the encrypted vault.
- **C L2:** Logs, request logs and model-bound prompts are covered; telemetry carries raw task URLs and code blocks receive raw secret values. — [skyvern/forge/log_redaction.py:36-45](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/log_redaction.py#L36-L45); [skyvern/forge/sdk/routes/agent_protocol.py:344](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/routes/agent_protocol.py#L344) (verified)
  - *To reach the next level:* Telemetry and code-block parameters are not redacted.
- **D L0:** Telemetry is on by default with a baked-in PostHog key and sends task URLs, which are user content, to a third party. — [skyvern/config.py:383-386](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/config.py#L383-L386); [skyvern/forge/sdk/routes/agent_protocol.py:344](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/routes/agent_protocol.py#L344) (verified)
  - *To reach the next level:* Telemetry is not opt-in and not content-free.
- **B L1:** Leaked secrets are long-lived website passwords, TOTP seeds and LLM keys, each scoped to its own service. — [skyvern/forge/sdk/workflow/context_manager.py:692-711](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/context_manager.py#L692-L711) (verified)
  - *To reach the next level:* Credentials are not short-lived or rotatable by design.
- **Cap:** none

### C9 Audit & traceability — 0.50 (high)

Every agent action is written to the database as a structured record (type, target element, reasoning, status, timestamps) together with screenshots, page HTML, LLM prompts and responses, and recordings, and code-block page calls are mapped onto the same action timeline. Records carry the organization but no separate human principal or approver, live in a database the server process itself can modify, and are written after the action runs; artifact capture is fire-and-forget. This is good for replaying a run, weaker as tamper-evident audit.

- **S L2:** Structured per-action records with status and timestamps, plus LLM and screenshot artifacts. — [skyvern/forge/agent.py:5118-5121](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/agent.py#L5118-L5121); [skyvern/forge/sdk/db/repositories/workflow_parameters.py:2069-2074](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/db/repositories/workflow_parameters.py#L2069-L2074) (verified)
  - *To reach the next level:* No actor attribution beyond the organization and no correlation of approver/principal.
- **C L2:** Agent actions and code-block page calls are recorded; block-level runs (HTTP, email) record outputs. — [skyvern/forge/sdk/workflow/models/code_block_recorder.py:109-115](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/models/code_block_recorder.py#L109-L115) (verified)
  - *To reach the next level:* Configuration changes and credential use are not recorded as audit events.
- **D L2:** On by default and stored outside any workspace, but in the same database the server process writes. — [skyvern/forge/agent.py:5118-5121](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/agent.py#L5118-L5121) (verified)
  - *To reach the next level:* Not written by a component isolated from the agent process.
- **B L2:** Each action record is awaited and persisted right after the action executes (errors surface), while artifact capture runs as a background task. — [skyvern/forge/agent.py:5010-5013](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/agent.py#L5010-L5013) (verified)
  - *To reach the next level:* No durable pre-action record and artifacts are best-effort, so a full trajectory is not guaranteed after a crash.
- **Cap:** none

### C10 Limits & kill switch — 0.45 (high)

Each task stops after 10 steps by default (each step with up to 5 retries), browser actions and code blocks have timeouts, and an optional per-organization budget can cap steps across a whole workflow run. There is no cost or token budget on the main agent loop and no default wall-clock limit for a run, and workflow loops may run up to 1000 iterations, each starting a task with its own step budget, with the loop count often driven by data extracted from pages. Cancelling marks the task in the database and is checked between steps, so in-flight actions finish.

- **S L2:** Step cap plus per-action and code-block timeouts are enforced in code; no cost cap and cooperative halt only. — [skyvern/config.py:176](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/config.py#L176); [skyvern/config.py:171](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/config.py#L171); [skyvern/forge/agent.py:10821](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/agent.py#L10821) (verified)
  - *To reach the next level:* No token/cost cap or wall-clock cap on the main loop.
- **C L2:** Limits apply per task and per tool call; the run-wide step budget across blocks is opt-in per organization. — [skyvern/forge/agent.py:10549-10550](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/agent.py#L10549-L10550) (verified)
  - *To reach the next level:* Loop iterations and child tasks do not count against a shared default budget.
- **D L2:** Sensible per-task defaults that the operator can override per request (max_steps header). — [skyvern/forge/sdk/workflow/models/block.py:780](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/workflow/models/block.py#L780) (verified)
  - *To reach the next level:* Loop iteration counts can be driven by page-extracted data, so the effective budget is not fixed.
- **B L1:** A workflow can run up to 1000 loop iterations of 10-step tasks with no spend ceiling; cancel only marks a DB row. — [skyvern/forge/agent.py:3694-3698](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/agent.py#L3694-L3698); [skyvern/forge/sdk/routes/agent_protocol.py:4001](https://github.com/Skyvern-AI/skyvern/blob/19bf399835a029b70a9fcec6dcea347a1c40c033/skyvern/forge/sdk/routes/agent_protocol.py#L4001) (verified)
  - *To reach the next level:* Ceilings are very large and stopping does not interrupt in-flight work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Every scraped web page (DOM, text, screenshots) enters the action prompt (skyvern/forge/prompts/skyvern/extract-action.j2:1) · [B] sensitive data/systems: Workflow-bound site credentials/TOTP resolved at execution and session cookies · [C] state change / egress: goto_url to any public host and form submission (skyvern/webeye/actions/handler.py:11351), send-email and HTTP blocks · Same default session? Yes

## Highest-impact improvements
1. Extend credential site-binding to every credential-release path. — C1 C L1→L2, +0.075 before caps (Playbook 4)
2. Make telemetry opt-in and drop task URLs from PostHog events. — C8 D L0→L2, +0.100 before caps (Playbook 4)
3. Default CODE_BLOCK_MODE to disabled/entitlement in OSS, or run code blocks in a hardened container without the server's environment. — C4 B L0→L2, +0.100 before caps (Playbook 3)
4. Expose firewall enrollment in OSS and default runs to an origin allowlist derived from the task URL, denying off-origin navigation and egress. — C5 B L0→L1, +0.050 before caps (Playbook 1)
5. Add an optional per-action human approval for submits, uploads, and off-origin navigation, showing the exact element, value placeholder and destination. — C2 S L0→L3, +0.225 before caps (Playbook 5)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The repository is very large (multi-thousand-line modules); review focused on the default v1 task engine, workflow blocks, code-block execution, credential handling, URL validation, telemetry and limits. Task v3, CUA engines, the workflow copilot, the MCP server, the browser extension and the frontend were reviewed only where they touch these controls.
- Cloud-only components referenced by the code (secure code-block runner, observation detector, firewall authority source) are not in the repo and were not credited.
- Docker Compose deployment (ENABLE_CODE_BLOCK=true, container) and Helm/k8s manifests were not scored; the pip quickstart that the README recommends was.
- No reviewer-steering text was found in AGENTS.md, CLAUDE.md, README.md or SECURITY.md.
