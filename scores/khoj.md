# Defense-in-Depth Score: Khoj

**Repo:** https://github.com/khoj-ai/khoj · **Commit:** `ae229ca894c0b80ad84664afcfdde523b5e87057` · **Reviewed:** 2026-10-03
**What it is:** Self-hostable AI second brain with custom agents, automations, deep research and code execution
**Category:** AI Assistants
**Scored configuration:** The shipped docker-compose.yml as documented in the setup guide, Terrarium code sandbox, SearxNG web search, telemetry on, operator and MCP not configured.
**Agent surface (default):** code execution yes · filesystem write no · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication opt-in

## Score: 3.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L2 | L0 | L0 | 0.23 | — | **0.23** | High |
| C2 | Approval gates | L0 | L0 | L0 | L2 | 0.10 | — | **0.10** | High |
| C3 | Tool & action scoping | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C4 | Code-execution isolation | L2 | L3 | L3 | L1 | 0.57 | — | **0.57** | High |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L1 | 0.23 | — | **0.23** | Medium |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C9 | Audit & traceability | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |


Khoj keeps model-written code out of its own process and its file tools only see the requesting user's documents, but nothing asks a human before any tool runs. The dominant risk is data leakage: a web page or document can steer the model into fetching an attacker URL that carries your notes or memories, and the web reader and chat renderer are not locked down either. The shipped compose file adds deployment risk: its access-control and credential defaults are not locked down, logs are verbose and unredacted, and telemetry is on by default.

## Critical gaps
- The server's single database-superuser identity exposes every user's data and every stored provider key if the authorization boundary fails; the shipped compose deployment's access-control and credential defaults are not locked down. (ASI03, T3, T9; C1) — [docker-compose.yml:67-68](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/docker-compose.yml#L67-L68)

## Criterion details

### C1 Identity & least privilege — 0.23 (high)

Khoj runs every tool with the server's own identity: one Postgres login (the database superuser in the shipped compose file), the operator's model and search API keys, and the server's network position. Built-in document, file and memory tools do scope their database queries to the requesting user, which is a real per-user check. Admin-configured MCP servers and the web reader are shared by all users and run with the server's full authority. The shipped docker-compose deployment's access-control and credential defaults are not locked down.

- **S L1:** A single broadly-scoped service identity (compose sets POSTGRES_USER=postgres, the database superuser) backs every tool; there is no per-tool or per-capability credential. — [docker-compose.yml:67-68](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/docker-compose.yml#L67-L68) (verified)
  - *To reach the next level:* No role-scoped database or API identity; read and write paths share the superuser login.
- **C L2:** Built-in file and memory tools filter by the requesting user in the query, but MCP servers are loaded globally for every user's research run and called with no per-user authorization. — [src/khoj/database/adapters/__init__.py:1894-1895](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/database/adapters/__init__.py#L1894-L1895); [src/khoj/routers/research.py:501-502](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/research.py#L501-L502); [src/khoj/database/adapters/__init__.py:2286](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/database/adapters/__init__.py#L2286) (verified)
  - *To reach the next level:* MCP tool calls and web fetches do not pass through a per-user authorization layer.
- **D L0:** The shipped compose deployment does not start from least privilege; its access-control and credential defaults are not locked down. (verified)
  - *To reach the next level:* Default deployment should start from least privilege with hardened access control and credentials.
- **B L0:** If the server's authorization layer fails, the attacker holds the database superuser over every user's notes, conversations and memories, all stored provider keys, and can register an MCP server path that the server launches with npx. — [docker-compose.yml:67-68](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/docker-compose.yml#L67-L68); [src/khoj/processor/tools/mcp.py:92-95](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/tools/mcp.py#L92-L95); [src/khoj/database/models/__init__.py:208](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/database/models/__init__.py#L208) (verified)
  - *To reach the next level:* Use a least-privilege DB role and separate per-tenant data so one compromise does not expose all users and all keys.
- **Cap:** none
- **Notes:** G1 does not apply: the per-user query scoping is always on; D is L0 because of the shipped compose deployment's access-control and credential defaults, not because the control is opt-in.

### C2 Approval gates — 0.10 (high)

There is no human approval step anywhere in Khoj's tool path. The model picks tools (web search, arbitrary web page reads, sandboxed Python, file and note search, admin-configured MCP tools, and the opt-in computer operator) and they run immediately. In the default configuration the built-in tools are mostly read-only, so the main unattended effects are outbound requests to model-chosen URLs and automatic memory writes. Any mutating MCP tool an admin adds would also run with no confirmation.

- **S L0:** No approval mechanism exists; a search for approval/confirmation code finds only an unrelated agent-listing TODO and a Twilio phone verification. — searched `rg -n -i 'approv|human_in_the_loop|require_confirmation'` in `src/khoj` → 2 hits (adapters:736 is a TODO about officially approved public agents; twilio.py:36 is phone OTP verification. Neither gates tools.) (verified)
  - *To reach the next level:* Add per-call human approval that shows the exact tool call and arguments.
- **C L0:** Every tool, including code execution and MCP tools, is executed directly from the research loop with no gate. — [src/khoj/routers/research.py:189-190](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/research.py#L189-L190); [src/khoj/routers/research.py:253-259](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/research.py#L253-L259) (verified)
  - *To reach the next level:* Route every tool call, including MCP and operator actions, through one gate.
- **D L0:** Approval is not available at all, so the default is no approval. — [src/khoj/routers/research.py:626-645](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/research.py#L626-L645) (verified)
  - *To reach the next level:* Ship approval on by default for consequential tools.
- **B L2:** Default tools are mostly read-only (note search, sandboxed Python without network, web reads); memory writes are reversible from the memory settings API, but model-chosen GET requests and admin-added MCP tools can have irreversible effects. — [src/khoj/utils/helpers.py:490-491](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/utils/helpers.py#L490-L491); [src/khoj/routers/api_memories.py:43](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/api_memories.py#L43); [src/khoj/processor/tools/online_search.py:494-497](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/tools/online_search.py#L494-L497) (verified)
  - *To reach the next level:* No previews or rollback for external actions; no bounds on how many URLs or MCP calls run per step.
- **Cap:** none

### C3 Tool & action scoping — 0.40 (high)

Khoj's file tools are narrow: view, list and regex-search operate only on the requesting user's indexed documents in the database, and regex patterns are compiled before use. The web reader is the opposite: it fetches any model-chosen URL, and its handling of internal destinations is not a strict boundary. The prompt tells the model to read only one page, but the code reads every URL it is given. All tools are enabled by default, though custom agents can be limited to a subset.

- **S L2:** File tools are DB queries scoped to the user with a compiled-regex check, but the web reader accepts any URL. — [src/khoj/routers/helpers.py:3205](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/helpers.py#L3205) (verified)
  - *To reach the next level:* Validate web-reader destinations against an allowlist or public-only policy.
- **C L2:** Most built-in tools are scoped or bounded, but the web reader and MCP tool arguments are passed through without validation, and the advertised per-step URL limit is only in the prompt. — [src/khoj/utils/helpers.py:517](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/utils/helpers.py#L517); [src/khoj/processor/tools/online_search.py:494-497](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/tools/online_search.py#L494-L497); [src/khoj/routers/research.py:259](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/research.py#L259) (verified)
  - *To reach the next level:* No shared validation layer covers MCP tools, and the web reader has no allowlist or count cap.
- **D L1:** The shipped compose enables web search, web reading and code execution for every chat; operators can drop a tool by unsetting its env var and agent creators can restrict input tools, but the default agent gets all. — [docker-compose.yml:76](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/docker-compose.yml#L76); [docker-compose.yml:80](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/docker-compose.yml#L80); [src/khoj/routers/research.py:365-366](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/research.py#L365-L366) (verified)
  - *To reach the next level:* Default to a read-only tool set with network fetch and code execution requiring explicit enabling.
- **B L1:** A misused web reader reaches any host its destination handling permits and a misused code tool runs arbitrary Python in the sandbox. (verified)
  - *To reach the next level:* Scope network reach to public hosts and bound the number of fetches per step.
- **Cap:** none

### C4 Code-execution isolation — 0.57 (high)

Model-written Python never runs inside the Khoj server process: it is posted over HTTP to a separate sandbox container (the Terrarium service in the shipped compose) or to E2B if an E2B key is set. If no sandbox is configured, the code tool is hidden rather than falling back to the host, which fails closed. The compose service is a stock container with no hardening flags, no resource limits and no network restriction, and it sits on the same Docker network as the Postgres database. Terrarium's WebAssembly isolation is described upstream but lives in an external, unpinned image, so this review could only credit the container boundary.

- **S L2:** Code goes to a separate container service; nothing in this repo hardens that container (no user, cap_drop, read_only, or security_opt), and the WASM boundary is inside an external image not reviewable here. — [docker-compose.yml:15-21](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/docker-compose.yml#L15-L21); [src/khoj/processor/tools/run_code.py:320-329](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/tools/run_code.py#L320-L329) (verified)
  - *To reach the next level:* Ship a hardened sandbox definition (non-root, dropped capabilities, no network, read-only root) or a kernel-separated sandbox by default.
- **C L3:** The only model-reachable code path is run_code, which always goes to E2B or Terrarium; the tool is withheld when no sandbox is configured, with no host fallback. — [src/khoj/processor/tools/run_code.py:224-229](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/tools/run_code.py#L224-L229); [src/khoj/routers/research.py:354](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/research.py#L354) (verified)
  - *To reach the next level:* Admin-added stdio MCP servers and the opt-in operator still run outside the sandbox.
- **D L3:** The sandbox is the default in compose and is selected purely by operator env vars that the model cannot change; removing them disables code execution instead of unsandboxing it. — [src/khoj/utils/helpers.py:784-787](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/utils/helpers.py#L784-L787); [docker-compose.yml:76](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/docker-compose.yml#L76) (verified)
  - *To reach the next level:* Sandbox definition is an unpinned :latest image; policy is not pinned or verified.
- **B L1:** If the in-container isolation breaks, the sandbox has unrestricted network on the compose network, where Postgres listens, and receives the user's files as input. — [docker-compose.yml:15-21](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/docker-compose.yml#L15-L21); [src/khoj/processor/tools/run_code.py:89-96](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/tools/run_code.py#L89-L96) (verified)
  - *To reach the next level:* Put the sandbox on an isolated network with egress denied and resource limits.
- **Cap:** none

### C5 Untrusted input blast radius — 0.30 (high)

Khoj reads untrusted web pages, search results, uploaded documents, MCP tool output and MCP tool descriptions, and in multi-user deployments other users' public agent personas. The only defenses are XML-style wrappers around tool results and an LLM 'safety' check on public agent personas that is not a strict boundary. A hijacked session holds the user's notes and memories and can send them out by asking the server to fetch an attacker URL; a further unattended egress path also exists. Nothing in default Khoj can take irreversible actions on external systems, so the worst case is unattended data leakage rather than destructive action.

- **S L1:** Tool results are wrapped in tags like <online_results>, and public agent personas go through an LLM safety prompt; both are detection or delimiting only. — [src/khoj/routers/research.py:683-684](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/research.py#L683-L684); [src/khoj/routers/helpers.py:334](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/helpers.py#L334) (verified)
  - *To reach the next level:* Disable or human-gate egress once untrusted content has entered the session.
- **C L1:** Wrappers apply to research tool results, but MCP tool descriptions go straight into the planner's tool list and memories are injected as a user-role message. — [src/khoj/routers/research.py:382](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/research.py#L382); [src/khoj/processor/conversation/utils.py:825-830](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/conversation/utils.py#L825-L830) (verified)
  - *To reach the next level:* Treat MCP descriptions, memories and other users' agent personas as untrusted data too.
- **D L2:** The wrappers are hard-coded and always on; nothing the agent reads can turn them off, but they are not a control the operator configures. — [src/khoj/routers/research.py:680-693](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/research.py#L680-L693) (verified)
  - *To reach the next level:* Defaults are not backed by a structural control that warns when disabled.
- **B L1:** An injected page can make the model fetch an attacker URL carrying the user's notes or memories with no human involved (a further egress path also exists); no default tool takes irreversible external actions. — [src/khoj/processor/tools/online_search.py:494-497](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/tools/online_search.py#L494-L497) (verified)
  - *To reach the next level:* Remove unattended egress so exfiltration needs approval.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.30 (high)

Long-term memory is on by default. After every chat turn an LLM pulls 'facts' from the latest exchange, which can include text echoed from web pages, and saves them with no validation or user confirmation. Saved memories are injected into later chats, including tool selection, as a user-role message about the user. Memories are stored per user and every query filters by the requesting user, and users can list, edit and delete them through the memory API. The server has no workspace-loaded config files.

- **S L1:** Memory writes are LLM-extracted and saved directly, with only an INFO log line; they are re-injected as user-role context. — [src/khoj/routers/helpers.py:1059-1062](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/helpers.py#L1059-L1062); [src/khoj/processor/conversation/utils.py:825-830](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/conversation/utils.py#L825-L830) (verified)
  - *To reach the next level:* Gate memory writes (user confirmation or source restrictions) and add expiry.
- **C L1:** Only the memory store has logging; public agent personas get an LLM check, and conversation history reuse has no control. — [src/khoj/routers/helpers.py:1060-1061](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/helpers.py#L1060-L1061) (verified)
  - *To reach the next level:* Extend controls to agent personas and conversation reuse with provenance tags.
- **D L2:** Memories are namespaced per user and enforced in every retrieval query; memory is on by default for new users. — [src/khoj/database/adapters/__init__.py:2343-2345](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/database/adapters/__init__.py#L2343-L2345); [src/khoj/database/models/__init__.py:516-519](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/database/models/__init__.py#L516-L519) (verified)
  - *To reach the next level:* The control mechanism is too weak for credit above one level over strength; retention limits are not on by default for searched memories.
- **B L1:** A poisoned memory persists across the user's sessions and is fed into tool selection; purge is possible but only if the user notices. — [src/khoj/routers/research.py:288](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/research.py#L288); [src/khoj/routers/api_memories.py:43](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/api_memories.py#L43) (verified)
  - *To reach the next level:* Poisoned memory should only influence text or need review before persisting.
- **Cap:** none

### C7 Third-party extensions — 0.23 (medium)

No third-party extension is enabled by default. An admin can add MCP servers in the admin panel; a bare package name is launched with npx and resolved to whatever version is current, with no pinning, hash check or re-approval when tool definitions change. Stdio servers run as child processes inside the Khoj server container as the same user. Only admins can add servers, and the repo has no workspace that could add them. Embedding models are downloaded from Hugging Face without trust_remote_code by default.

- **S L1:** Admin-chosen MCP servers are launched with npx or python/node on the given path with no version pin enforced and no integrity check. — [src/khoj/processor/tools/mcp.py:92-103](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/tools/mcp.py#L92-L103) (verified)
  - *To reach the next level:* Require pinned versions and verify a hash or signature before launch.
- **C L0:** No extension type (MCP stdio, MCP SSE, downloaded models) is verified. — searched `rg -n 'sha256|integrity|signature'` in `src/khoj/processor/tools` → 0 hits (No verification code near the MCP client or other tool loaders.) (verified)
  - *To reach the next level:* Verify at least MCP launches.
- **D L2:** MCP servers exist only if an admin adds them in the Django admin, entering name and path; nothing shows what will run or with what permissions. — [src/khoj/database/admin.py:189-190](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/database/admin.py#L189-L190); [src/khoj/database/models/__init__.py:846-849](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/database/models/__init__.py#L846-L849) (verified)
  - *To reach the next level:* Show the exact command and permissions at add time and re-approve on change.
- **B L1:** Stdio servers run in a separate process in the same container and user as the server; env=None means the MCP SDK's default reduced environment (inferred from SDK behaviour), but the same user can still read the server's files and process environment. — [src/khoj/processor/tools/mcp.py:105](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/tools/mcp.py#L105) (inferred)
  - *To reach the next level:* Run each MCP server sandboxed with its own credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.10 (high)

The shipped deployment's credential defaults are not locked down. Provider API keys are stored as plaintext database fields. There is no redaction anywhere: the file log handler always records DEBUG output, compose starts with -vv, and every conversation turn and every executed code snippet with its output is logged at INFO. Usage telemetry is on by default and sends content-free metadata (user UUID, host, referrer, command, agent slug) to a Khoj-run endpoint. Secrets are not routinely placed in model context.

- **S L0:** Credential defaults are not locked down; no masking exists. (verified)
  - *To reach the next level:* Harden credential defaults and add redaction in logging.
- **C L0:** No path (logs, transcripts, telemetry, subprocess env) is redacted. — searched `rg -n -i 'redact|SecretStr|mask_secret'` in `src/khoj` → 3 hits (All three hits are image placeholders ('redacted for space'), not secret redaction.) (verified)
  - *To reach the next level:* Redact secrets and sensitive content on at least the log path.
- **D L1:** Telemetry is on unless KHOJ_TELEMETRY_DISABLE is set and is content-free; the file log handler is always DEBUG and compose sets -vv, logging full chat turns and code output. — [src/khoj/utils/state.py:33](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/utils/state.py#L33); [src/khoj/main.py:144-145](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/main.py#L144-L145); [src/khoj/processor/tools/run_code.py:107](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/tools/run_code.py#L107); [src/khoj/processor/conversation/utils.py:627](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/conversation/utils.py#L627) (verified)
  - *To reach the next level:* Make telemetry opt-in and keep default logs free of conversation content.
- **B L1:** Leaked material is long-lived provider keys and the DB superuser password; the model and sandbox do not receive them directly. — [src/khoj/database/models/__init__.py:208](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/database/models/__init__.py#L208) (verified)
  - *To reach the next level:* Use scoped, rotatable keys.
- **Cap:** none

### C9 Audit & traceability — 0.38 (high)

Khoj stores each chat turn in its database with the user and agent, and keeps code and online-search context for completed turns. Research-mode tool calls, including MCP calls with their arguments, appear only as INFO log lines: the structured research and operator context is saved only when a turn is interrupted and dropped when it completes. Records are written by the server, outside anything the model can touch, but only at the end of a turn. There are no approvals to record, no actor chain beyond user and agent, and no tamper evidence.

- **S L1:** Research tool calls are recorded as unstructured log lines; the structured researchContext is discarded on successful completion. — [src/khoj/routers/research.py:461-463](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/research.py#L461-L463); [src/khoj/processor/conversation/utils.py:579-580](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/conversation/utils.py#L579-L580) (verified)
  - *To reach the next level:* Persist a structured record of every tool call with arguments, status and timestamps.
- **C L2:** All tool calls, including MCP calls, reach the log line; memory writes are also logged, but config changes and credential use are not. — [src/khoj/routers/research.py:461-463](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/research.py#L461-L463); [src/khoj/routers/helpers.py:1060-1061](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/helpers.py#L1060-L1061) (verified)
  - *To reach the next level:* Record approvals and config/credential events in the same record.
- **D L2:** Logging is on by default and written by the server, which the model cannot control; users can still delete conversations. — [src/khoj/main.py:144-146](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/main.py#L144-L146) (verified)
  - *To reach the next level:* Store records append-only or off-host.
- **B L1:** Conversation records are written at the end of the turn; logging failures are silent. — [src/khoj/processor/conversation/utils.py:603-609](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/conversation/utils.py#L603-L609) (verified)
  - *To reach the next level:* Write records durably per action and surface failures.
- **Cap:** none

### C10 Limits & kill switch — 0.50 (high)

Research mode stops after 5 iterations by default (KHOJ_RESEARCH_ITERATIONS), and individual tools have timeouts: 30 seconds for sandboxed code, 60 seconds for page fetches, and HTTP timeouts on model calls. There is no token or cost cap, and per-user chat rate limits are skipped whenever billing is not configured, which is the self-hosted default. A single iteration can fan out to any number of parallel tool calls. Stopping a chat cancels the asyncio task, and a code timeout calls the sandbox's stop endpoint. User-created scheduled automations keep running until deleted.

- **S L2:** An iteration cap plus per-execution timeouts are enforced in code; no token/cost cap and no wall-clock cap per run. — [src/khoj/routers/research.py:498](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/research.py#L498); [src/khoj/processor/tools/run_code.py:329](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/tools/run_code.py#L329); [src/khoj/processor/tools/online_search.py:53](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/tools/online_search.py#L53) (verified)
  - *To reach the next level:* Add token/cost and wall-clock caps and rate limits on side-effecting tools.
- **C L2:** The loop cap and tool timeouts apply, but parallel tool calls per iteration are unbounded and opt-in operator runs get their own 100-step budget. — [src/khoj/routers/research.py:626-645](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/research.py#L626-L645); [src/khoj/processor/operator/__init__.py:70](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/operator/__init__.py#L70) (verified)
  - *To reach the next level:* Count all spawned work against one budget and cap fan-out per step.
- **D L2:** Defaults are modest and only operator env vars change them, but chat rate limits are disabled whenever billing is not configured. — [src/khoj/routers/helpers.py:2111-2113](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/helpers.py#L2111-L2113) (verified)
  - *To reach the next level:* Keep rate limits on in self-hosted deployments.
- **B L2:** Stopping cancels the running asyncio task and the research loop checks cancellation between iterations; sandbox code runs until its timeout, and scheduled automations keep running. — [src/khoj/routers/research.py:511-515](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/research.py#L511-L515); [src/khoj/routers/api_chat.py:1567-1569](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/routers/api_chat.py#L1567-L1569); [src/khoj/processor/tools/run_code.py:113](https://github.com/khoj-ai/khoj/blob/ae229ca894c0b80ad84664afcfdde523b5e87057/src/khoj/processor/tools/run_code.py#L113) (verified)
  - *To reach the next level:* Cancel in-flight sandbox execution on stop and add provider-side spend ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Web pages fetched by read_webpage_at_url (src/khoj/processor/tools/online_search.py:572-578) and MCP results (src/khoj/routers/research.py:259) · [B] sensitive data/systems: User notes via view_file/semantic search (src/khoj/routers/helpers.py:3205) and long-term memories (src/khoj/processor/conversation/utils.py:825) · [C] state change / egress: Model-chosen URL fetches (src/khoj/processor/tools/online_search.py:494-497) and automatic memory writes (src/khoj/routers/helpers.py:1062); a further egress path also exists · Same default session? Yes

## Highest-impact improvements
1. Harden the shipped docker-compose deployment's access-control and credential defaults. — C1 D L0→L2, +0.100 before caps (Playbook 4)
2. Harden web-reader destination validation and enforce max_webpages_to_read in code. — C3 S L2→L3, +0.075 before caps (Playbook 3)
3. Once web or MCP content has been read, require user approval for further model-chosen URL fetches and close other unattended egress paths. — C5 S L1→L2, +0.075 before caps (Playbook 1)
4. Add a per-call approval prompt for code execution, MCP tools and web fetches that shows the exact arguments. — C2 S L0→L3, +0.225 before caps (Playbook 5)
5. Persist the structured researchContext (tool name, arguments, status, timestamp) for completed turns, not only interrupted ones. — C9 S L1→L2, +0.075 before caps

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The Terrarium sandbox, SearxNG and khoj-computer images are external (unpinned :latest) and were not reviewed; C4 credits only what docker-compose.yml shows.
- MCP subprocess environment handling (env=None) relies on the MCP Python SDK's default-environment behaviour, which was inferred, not read.
- Mobile, desktop, Obsidian and Emacs clients and the hosted app.khoj.dev configuration were not examined; the opt-in computer/browser operator was reviewed only enough to confirm it is off by default.
- No reviewer-directed prompt-injection text was found in the repository.
