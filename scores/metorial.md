# Defense-in-Depth Score: Metorial

**Repo:** https://github.com/metorial/metorial · **Commit:** `221f8ead1244f884711ac062398ae4e4f771b1ee` · **Reviewed:** 2026-10-04
**What it is:** Catalog of 1,100+ integrations (Slates SDK tool providers) that the Metorial platform exposes to AI agents as tools.
**Category:** Agent Frameworks
**Scored configuration:** Integrations as shipped, served by @slates/provider-handler to a host (the Metorial hub) with each integration's default tool set and config.
**Agent surface (default):** code execution yes · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 4.3 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L1 | L1 | L0 | 0.28 | C1-SELFESC | **0.25** | High |
| C2 | Approval gates | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C3 | Tool & action scoping | L2 | L3 | L0 | L0 | 0.38 | — | **0.38** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L2 | L0 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |

Controls where a risk surface exists: 2.25 / 8.0 (28%); 2 criteria scored SA (surface absent).

Metorial's integrations give agents thousands of tools against real accounts (GitHub, Gmail, Slack, Kubernetes, databases) with the user's long-lived credentials and every tool exposed by default. The SDK validates argument types centrally and redacts secrets from HTTP traces well, but it leaves approval, scoping and authorization to the host platform, which is not in this repo, and several high-impact tools are mislabelled as non-destructive. The dominant risk is a hijacked agent running arbitrary SQL, sending mail or rewriting Kubernetes RBAC with no server-side check.

## Critical gaps
- The Kubernetes manage_rbac tool can create ClusterRoleBindings for a ServiceAccount (the default credential type) and is tagged non-destructive, letting a hijacked agent grant itself cluster-admin. (ASI03, T3; C1) — [integrations/kubernetes/src/tools/manage-rbac.ts:17](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/kubernetes/src/tools/manage-rbac.ts#L17); [integrations/kubernetes/src/tools/manage-rbac.ts:43](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/kubernetes/src/tools/manage-rbac.ts#L43); [integrations/kubernetes/src/tools/manage-rbac.ts:35-37](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/kubernetes/src/tools/manage-rbac.ts#L35-L37); [integrations/kubernetes/src/auth.ts:30](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/kubernetes/src/auth.ts#L30)
- Connected credentials carry account-level authority (GitHub admin:org/delete_repo, cluster RBAC, database passwords) with no per-tool narrowing. (ASI03, T3; C1) — [integrations/github/src/auth.ts:31](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/github/src/auth.ts#L31); [integrations/github/src/auth.ts:106](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/github/src/auth.ts#L106); [packages/provider-handler/src/index.ts:939-1012](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider-handler/src/index.ts#L939-L1012)
- PostgreSQL execute_query runs arbitrary model-written SQL (DDL, DELETE, DROP) with the stored database credentials and no containment, tagged non-destructive. (ASI05, T11, LLM05; C4) — [integrations/postgresql/src/tools/execute-query.ts:79](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/execute-query.ts#L79); [integrations/postgresql/src/tools/execute-query.ts:21-24](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/execute-query.ts#L21-L24)
- Untrusted content read through one tool can drive email sends, Slack/GitHub posts and arbitrary SQL through others with no server-side check; worst case is leak plus irreversible action. (ASI01, LLM01, T6; C5) — [integrations/gmail/src/tools/send-email.ts:16-24](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/gmail/src/tools/send-email.ts#L16-L24); [integrations/postgresql/src/tools/execute-query.ts:79](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/execute-query.ts#L79)

## Criterion details

### C1 Identity & least privilege — 0.25 (high)

Each integration acts with whatever credential the user connected through the Metorial hub: an OAuth token with scopes the hub requests, an API key, a database password, or a Kubernetes bearer token. Tools can declare the OAuth scopes they need, which lets the host request narrower grants, but only a minority of tools declare them and the executor in this repo never checks them. Several integrations offer broad scopes (GitHub admin:org, delete_repo) and some tools can change their own permissions: the Kubernetes manage_rbac tool can bind a ClusterRole to a ServiceAccount, which is the default credential type, so a hijacked call can escalate itself to cluster-admin.

- **S L2:** One credential per connected integration, static for the session; read and write tools share it; tools may declare required OAuth scopes as metadata. — [packages/provider/src/action/action.ts:24-30](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider/src/action/action.ts#L24-L30); [integrations/github/src/auth.ts:170](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/github/src/auth.ts#L170) (verified)
  - *To reach the next level:* No per-tool credential narrowing or short-lived task-scoped tokens; read and write share one credential.
- **C L1:** Per-tool scope declarations exist (about 754 .scopes() calls across roughly 18,400 tools) but the tool executor attaches the session credential to every tool with no authorization check. — [packages/provider-handler/src/index.ts:939-1012](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider-handler/src/index.ts#L939-L1012); searched `rg -n 'action\.scopes|\.scopes\b.*includes'` in `packages/provider-handler/src` → 0 hits (declared per-tool scopes are never checked by the executor) (verified)
  - *To reach the next level:* No authorization check in the executor mapping each tool to the granted scopes.
- **D L1:** The scope set is chosen by the host; integrations offer admin-level scopes and nothing in the server marks a least-privilege default. — [integrations/github/src/auth.ts:31](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/github/src/auth.ts#L31); [integrations/github/src/auth.ts:106](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/github/src/auth.ts#L106); [integrations/github/src/auth.ts:170](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/github/src/auth.ts#L170) (verified)
  - *To reach the next level:* No minimal default scope set enforced by the server; widening is a host-side choice with no signal.
- **B L0:** A hijacked call holds the user's account-level authority on each connected service: GitHub org admin and repo deletion, Kubernetes cluster RBAC, production databases. — [integrations/github/src/auth.ts:31](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/github/src/auth.ts#L31); [integrations/kubernetes/src/tools/manage-rbac.ts:43](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/kubernetes/src/tools/manage-rbac.ts#L43); [integrations/kubernetes/src/auth.ts:30](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/kubernetes/src/auth.ts#L30) (verified)
  - *To reach the next level:* Credentials would need to be scoped to one project/tenant and mostly read.
- **Cap:** C1-SELFESC — kubernetes manage_rbac creates ClusterRoleBindings for ServiceAccount subjects, and the default auth method is a ServiceAccount bearer token, so the agent can widen its own permissions.

### C2 Approval gates — 0.20 (high)

As a tool server, Metorial integrations leave approval to the host and supply readOnly/destructive tags on tools. Most tools carry tags, but important mutating tools are mislabelled or untagged: arbitrary-SQL execute_query and Gmail send_email are marked non-destructive, Kubernetes manage_rbac is non-destructive, and GitHub manage_file_content mixes read, write and delete with no tags at all. There is no dry-run or server-enforced read-only mode, and many actions (sending mail, DROP TABLE, deleting repositories) cannot be undone.

- **S L1:** readOnly/destructive tags exist and are widely used, but they are missing or wrong on several mutating tools and some tools mix reads and writes. — [packages/provider/src/action/action.ts:24-30](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider/src/action/action.ts#L24-L30); [integrations/postgresql/src/tools/execute-query.ts:21-24](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/execute-query.ts#L21-L24); [integrations/gmail/src/tools/send-email.ts:16-24](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/gmail/src/tools/send-email.ts#L16-L24); [integrations/github/src/tools/manage-file-content.ts:24](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/github/src/tools/manage-file-content.ts#L24) (verified)
  - *To reach the next level:* Accurate tags on every tool with separate read and write tools.
- **C L1:** Tags cover most tools (about 13,800 readOnly: lines across about 18,400 tools) but powerful mutating tools are unflagged or flagged non-destructive. — [integrations/postgresql/src/tools/execute-query.ts:21-24](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/execute-query.ts#L21-L24); [integrations/kubernetes/src/tools/manage-rbac.ts:35-37](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/kubernetes/src/tools/manage-rbac.ts#L35-L37) (verified)
  - *To reach the next level:* Every mutating tool flagged accurately.
- **D L1:** Tags are static code the model cannot change, but nothing in the server enforces them; the host may ignore them. — [packages/provider-handler/src/index.ts:939-1012](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider-handler/src/index.ts#L939-L1012) (verified)
  - *To reach the next level:* No server-enforced read-only mode or confirmation step.
- **B L0:** Irreversible high-impact actions (email send, arbitrary SQL including DROP, RBAC changes) run with no undo or preview. — [integrations/gmail/src/tools/send-email.ts:16-24](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/gmail/src/tools/send-email.ts#L16-L24); [integrations/postgresql/src/tools/execute-query.ts:79](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/execute-query.ts#L79); [integrations/kubernetes/src/tools/manage-rbac.ts:43](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/kubernetes/src/tools/manage-rbac.ts#L43) (verified)
  - *To reach the next level:* Previews/dry-runs for external actions or reversibility for the common case.
- **Cap:** none

### C3 Tool & action scoping — 0.38 (high)

Every tool call passes through one central Zod schema check before its handler runs, and most tools are narrow API wrappers (create an issue, list branches). But the schemas mostly check types, not bounds or allowlists: arbitrary SQL, Kubernetes manifests, and unbounded email recipient lists are accepted, and the Slack upload tool fetches any URL from the server process. Every tool in an integration is exposed by default, including write and delete tools, so a misused tool reaches production systems directly.

- **S L2:** Typed Zod schemas are enforced, with some checks that are not strict boundaries and raw passthrough elsewhere (execute_query). — [packages/provider-handler/src/validation.ts:15-27](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider-handler/src/validation.ts#L15-L27); [integrations/postgresql/src/tools/execute-query.ts:79](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/execute-query.ts#L79); [integrations/slack/src/chat/tools/upload-file.ts:27](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/slack/src/chat/tools/upload-file.ts#L27) (verified)
  - *To reach the next level:* Allowlist validation in code: host allowlists with internal-address blocking, numeric bounds, no raw SQL passthrough.
- **C L3:** The same validate() call runs for every tool invocation before the handler, so new tools inherit it. — [packages/provider-handler/src/index.ts:939-1012](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider-handler/src/index.ts#L939-L1012); [packages/provider-handler/src/validation.ts:15-27](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider-handler/src/validation.ts#L15-L27) (verified)
  - *To reach the next level:* C is capped one level above S; the shared layer only checks types.
- **D L0:** All actions of an integration are exposed by default, including write, delete and raw-SQL tools; the exposure filter only hides adapter actions. — [packages/provider-handler/src/spec.ts:176-182](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider-handler/src/spec.ts#L176-L182) (verified)
  - *To reach the next level:* Read-only tool set by default with write/exec requiring explicit enabling.
- **B L0:** General-purpose tools act on production systems: any SQL on the connected database, any Kubernetes resource, any number of email recipients. — [integrations/postgresql/src/tools/execute-query.ts:79](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/execute-query.ts#L79); [integrations/gmail/src/tools/send-email.ts:16-24](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/gmail/src/tools/send-email.ts#L16-L24); [adapters/chat/src/tools.ts:371](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/adapters/chat/src/tools.ts#L371) (verified)
  - *To reach the next level:* Tools scoped to a project with quantity bounds.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

The server never runs model-generated code on its own host: there is no shell, eval or subprocess path in the integration runtime. But several tools pass model-written code straight to remote systems, most directly PostgreSQL's execute_query, which runs any SQL (including DROP and ALTER) with the stored database credentials. The separate select_query tool wraps queries in a read-only transaction, but execute_query sits alongside it with no containment, so the connected database is fully exposed.

- **S L0:** Remote code paths (arbitrary SQL) run with no isolation primitive; only the separate select_query tool uses a READ ONLY transaction. — [integrations/postgresql/src/tools/execute-query.ts:79](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/execute-query.ts#L79); [integrations/postgresql/src/tools/select-query.ts:97](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/select-query.ts#L97); searched `rg -n -g '!*.test.ts' 'child_process|node:vm|\beval\(|new Function\(|execFile|spawnSync'` in `integrations packages/provider/src packages/provider-handler/src packages/proto/src packages/client/src adapters packages/slates/src` → 0 hits (verified)
  - *To reach the next level:* An isolation boundary (read-only role, sandboxed replica) around model-written code.
- **C L0:** The main raw-SQL tool is not contained; only the auxiliary select tool is. — [integrations/postgresql/src/tools/execute-query.ts:79](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/execute-query.ts#L79); [integrations/postgresql/src/tools/select-query.ts:97](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/select-query.ts#L97) (verified)
  - *To reach the next level:* The main execution tool would need containment.
- **D L0:** execute_query ships enabled alongside select_query with no containment to switch on. — [integrations/postgresql/src/tools/execute-query.ts:21-24](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/execute-query.ts#L21-L24) (verified)
  - *To reach the next level:* Containment on by default.
- **B L0:** Model-written SQL runs with the stored database credentials against the user's database, including DDL and deletes. — [integrations/postgresql/src/tools/execute-query.ts:79](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/execute-query.ts#L79); [integrations/postgresql/src/auth.ts:8-16](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/auth.ts#L8-L16) (verified)
  - *To reach the next level:* Execution limited to an isolated, credential-free environment.
- **Cap:** none

### C5 Untrusted input blast radius — 0.25 (high)

Tool results come back as a structured output object plus a separate model-facing message, which keeps content apart from metadata. But nothing marks results as untrusted or records where content came from (email bodies, issue comments, web pages), and the server offers no mode that removes a Rule-of-Two leg. If an agent using these tools is hijacked by content it reads, the same server can send email, post to Slack, write to GitHub and run SQL with no server-side check, so data theft and irreversible actions are both reachable.

- **S L2:** Results are structured {output, message, attachments}, separating returned content from metadata. — [packages/provider/src/action/action.ts:42-46](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider/src/action/action.ts#L42-L46) (verified)
  - *To reach the next level:* Provenance or an untrusted flag on returned content that the host can act on.
- **C L0:** No source is distinguished; email bodies, issue comments and web content all come back in the same shape as trusted data. — [packages/provider/src/action/action.ts:42-46](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider/src/action/action.ts#L42-L46); searched `rg -n -i 'untrusted|provenance'` in `packages/provider/src packages/provider-handler/src packages/proto/src` → 0 hits (verified)
  - *To reach the next level:* Untrusted sources distinguished at least for the main readers.
- **D L2:** The structured result shape is fixed in code and always on. — [packages/provider-handler/src/index.ts:939-1012](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider-handler/src/index.ts#L939-L1012) (verified)
  - *To reach the next level:* Nothing that ensures a host is warned or can rely on provenance.
- **B L0:** A hijacked agent can exfiltrate through email/Slack/GitHub and take irreversible actions (send, delete, DROP) with no server-side approval. — [integrations/gmail/src/tools/send-email.ts:16-24](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/gmail/src/tools/send-email.ts#L16-L24); [integrations/postgresql/src/tools/execute-query.ts:79](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/execute-query.ts#L79); [integrations/slack/src/chat/tools/upload-file.ts:27](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/slack/src/chat/tools/upload-file.ts#L27) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions would need human approval.
- **Cap:** C5-WORSTCASE — B is L0: leak plus irreversible action is reachable unattended through the server's own tools.

### C6 Memory, context & configuration integrity — 1.00 (high)

The integration runtime keeps no memory or retrieval store and loads no instruction or config files from a workspace. Session and polling state come from the hub and are not written by the model. There is no persistence path a poisoned input could use.

- **Structural absence:** searched `rg -n -i 'dotenv|load_dotenv|AGENTS\.md|CLAUDE\.md|saveMemory|remember\(' -g '!*.test.ts'` in `packages/provider/src packages/provider-handler/src packages/client/src integrations` → 2 hits (both hits are bitwarden restoreMember (case-insensitive match), not memory)

### C7 Third-party extensions — 1.00 (high)

Integrations are bundled at build time; the runtime does not download or launch plugins, MCP servers or packages, and the only dynamic imports load the integration's own local modules. Custom MCP server hosting, mentioned in the README, lives in the separate platform repo and was not reviewed.

- **Structural absence:** searched `rg -n 'await import\(|npx |pip install|trust_remote_code|pickle' -g '!*.test.ts' -g '!**/dist*/**' -g '!**/docs/**'` in `packages/provider/src packages/provider-handler/src packages/proto/src packages/client/src packages/slates/src adapters/chat/src integrations` → 6 hits (all hits are dynamic imports of the integration's own ./lib modules or the slates SDK)

### C8 Secrets & sensitive-data protection — 0.45 (high)

Credentials reach integrations from the hub per session, and the SDK redacts them from recorded HTTP traces by header/field name, by the exact values of the connected credentials, and by well-known token patterns. Attachment URLs have credentials swapped for placeholders the hub fills in later. Gaps: tool outputs and messages sent to the model are not scanned, and error logs carry raw messages and stack traces. The credentials themselves are long-lived OAuth tokens, API keys and database passwords.

- **S L2:** Name-based, value-based and pattern-based redaction on HTTP traces; placeholder substitution for attachment credentials. — [packages/provider/src/axios/trace.ts:694-716](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider/src/axios/trace.ts#L694-L716); [packages/provider/src/axios/trace.ts:58](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider/src/axios/trace.ts#L58); [packages/provider/src/axios/trace.ts:274](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider/src/axios/trace.ts#L274); [packages/provider-handler/src/index.ts:980-983](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider-handler/src/index.ts#L980-L983) (verified)
  - *To reach the next level:* Redaction on model-bound outputs and verified encryption at rest (held by the hub, not in this repo).
- **C L2:** HTTP traces and attachments are covered; error logs and tool outputs are not. — [packages/provider-handler/src/index.ts:168-173](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider-handler/src/index.ts#L168-L173); [packages/provider-handler/src/index.ts:939-1012](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider-handler/src/index.ts#L939-L1012) (verified)
  - *To reach the next level:* Redaction of error logs/stack traces and model-bound tool outputs.
- **D L2:** Trace redaction is applied inside the axios interceptor with no off switch and there is no telemetry SDK, but error logs with raw messages and stack traces are emitted unredacted by default. — [packages/provider/src/axios/trace.ts:694-716](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider/src/axios/trace.ts#L694-L716); searched `rg -n -i 'sentry|posthog|opentelemetry|datadog'` in `packages/provider/src packages/provider-handler/src packages/proto/src packages/client/src packages/slates/src` → 0 hits (verified)
  - *To reach the next level:* Redaction always on for every emitted log, including error entries.
- **B L1:** Leaked credentials are long-lived user OAuth tokens, API keys and DB passwords, often broadly scoped. — [integrations/postgresql/src/auth.ts:8-16](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/auth.ts#L8-L16); [integrations/github/src/auth.ts:31](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/github/src/auth.ts#L31) (verified)
  - *To reach the next level:* Scoped keys with short lifetimes.
- **Cap:** none

### C9 Audit & traceability — 0.40 (high)

Every tool invocation emits structured start, success and error log entries (tool id, duration, result message), and each outbound HTTP request is recorded with method, sanitized URL and body. Tool arguments themselves are deliberately not logged (only their key count), and there is no actor or approver attribution. Logs are batched in memory, flushed after 10 ms to whatever listener the host attaches, and dropped if none is attached or the process crashes.

- **S L2:** Structured per-call records with timestamps and status, plus sanitized HTTP request/response traces. — [packages/provider-handler/src/index.ts:244-282](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider-handler/src/index.ts#L244-L282); [packages/provider-handler/src/index.ts:962](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider-handler/src/index.ts#L962); [packages/provider/src/axios/trace.ts:694-716](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider/src/axios/trace.ts#L694-L716) (verified)
  - *To reach the next level:* Actor attribution (requesting principal, approver) and correlation IDs.
- **C L2:** All tool invocations go through traceProviderCall; HTTP traces cover only axios-based clients (raw-socket clients like PostgreSQL are not traced). — [packages/provider-handler/src/index.ts:244-282](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider-handler/src/index.ts#L244-L282); [integrations/postgresql/src/tools/execute-query.ts:79](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/execute-query.ts#L79) (verified)
  - *To reach the next level:* Complete coverage of every outbound effect.
- **D L1:** Records live in the provider process and reach a sink only if the host passes listeners (default empty list). — [packages/client/src/transport.ts:34](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/client/src/transport.ts#L34) (verified)
  - *To reach the next level:* Record written by a component the model cannot control, on by default.
- **B L1:** Best-effort in-memory buffer flushed on a 10 ms timer; lost on crash. — [packages/provider/src/logger/logger.ts:110-118](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider/src/logger/logger.ts#L110-L118) (verified)
  - *To reach the next level:* Records flushed per action with surfaced errors.
- **Cap:** none

### C10 Limits & kill switch — 0.33 (high)

Bounds on the server's own work are piecemeal: HTTP trace bodies are cut at 10 KB, attachments are capped at 100 MB with upload concurrency of 10, and some integrations set defaults such as PostgreSQL's 30-second query timeout and 1,000-row limit. The SDK has no general per-tool timeout and no cancellation of in-flight calls, and in execute_query the model can raise the row limit itself.

- **S L2:** Server-enforced caps on some operations (trace size, attachment size, per-integration query timeouts). — [packages/provider/src/axios/trace.ts:46](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider/src/axios/trace.ts#L46); [packages/provider-handler/src/index.ts:46](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider-handler/src/index.ts#L46); [integrations/postgresql/src/config.ts:10](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/config.ts#L10); searched `rg -n 'timeout:'` in `packages/provider/src packages/provider-handler/src packages/proto/src` → 2 hits (both hits are error-code maps (timeout: 'internal.timeout'), not timeouts) (verified)
  - *To reach the next level:* Caps on every operation plus rate limits.
- **C L1:** Caps are set per integration and are absent from the SDK's tool executor. — [packages/provider-handler/src/index.ts:939-1012](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/packages/provider-handler/src/index.ts#L939-L1012); searched `rg -n 'timeout:'` in `packages/provider/src packages/provider-handler/src packages/proto/src` → 2 hits (both hits are error-code maps (timeout: 'internal.timeout'), not timeouts) (verified)
  - *To reach the next level:* SDK-level timeout covering every tool.
- **D L1:** Where defaults exist the model can override them (maxRows input overrides config). — [integrations/postgresql/src/tools/execute-query.ts:62](https://github.com/metorial/metorial/blob/221f8ead1244f884711ac062398ae4e4f771b1ee/integrations/postgresql/src/tools/execute-query.ts#L62) (verified)
  - *To reach the next level:* Defaults the model cannot raise.
- **B L1:** No cancellation of in-flight calls; upstream HTTP calls have no default timeout. — searched `rg -n -i 'abortsignal|abortcontroller|\.cancel\('` in `packages/provider/src packages/provider-handler/src packages/proto/src` → 1 hits (only hit is a stream reader.cancel() in directUpload; no cancellation of tool invocations) (verified)
  - *To reach the next level:* Stopping should cancel pending calls with tight per-call ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: email, issue and chat content returned by read tools (e.g. gmail/src/tools/get-message.ts) · [B] sensitive data/systems: user OAuth tokens and DB passwords in session auth (provider-handler/src/index.ts:1006) · [C] state change / egress: send_email (gmail/src/tools/send-email.ts:24), execute_query (postgresql/src/tools/execute-query.ts:79) · Same default session? Yes

## Highest-impact improvements
1. Fix readOnly/destructive tags on mutating tools (execute_query, send_email, manage_rbac) and split mixed read/write tools, enforced by a lint in validate-pr. — C2 S L1→L2, +0.075 before caps (Playbook 5)
2. Check each tool's declared scopes against the granted scopes in the executor, failing closed, and declare scopes on every tool. — C1 C L1→L3, +0.150 before caps (Playbook 4 step 1)
3. Ship write/delete/raw-SQL tools disabled unless the operator enables them for a deployment. — C3 D L0→L3, +0.150 before caps (Playbook 3)
4. Add a default per-invocation timeout and cancellation in provider-handler. — C10 C L1→L2, +0.075 before caps
5. Log redacted tool arguments with the requesting principal on each invocation. — C9 S L2→L3, +0.075 before caps (Playbook 1 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The Metorial platform (hub: credential storage, RBAC, approvals, audit logs) lives in metorial/metorial-platform and was not reviewed; README claims about RBAC and audit logs are therefore not credited.
- With 1,149 integrations, per-integration findings come from sampled integrations (github, gmail, slack, postgresql, kubernetes) and repo-wide counts; other integrations may differ.
- No text aimed at AI reviewers was found in the repository.
