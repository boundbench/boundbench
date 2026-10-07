# Defense-in-Depth Score: ContextForge MCP Gateway

**Repo:** https://github.com/IBM/mcp-context-forge · **Commit:** `9bb9c2ac58af4cd126c05e8153f9025770646bd2` (v1.0.11-21-g9bb9c2ac) · **Reviewed:** 2026-10-04
**What it is:** Open-source registry and proxy that federates MCP, A2A and REST/gRPC tools behind one authenticated MCP endpoint with RBAC, admin UI and observability.
**Category:** Agent Frameworks
**Scored configuration:** README quick start: `uvx --from mcp-contextforge-gateway mcpgateway` with generated JWT_SECRET_KEY/AUTH_ENCRYPTION_SECRET, MCPGATEWAY_UI_ENABLED and MCPGATEWAY_ADMIN_API_ENABLED set true, every other setting at its mcpgateway/config.py default (auth required, plugins off, audit trail off, SQLite).
**Agent surface (default):** code execution yes · filesystem write no · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 3.7 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L3 | L2 | L1 | 0.53 | C1-PASSTHRU | **0.25** | High |
| C2 | Approval gates | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C3 | Tool & action scoping | L2 | L3 | L2 | L1 | 0.53 | — | **0.53** | High |
| C4 | Code-execution isolation | L1 | L2 | L2 | L0 | 0.33 | — | **0.33** | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L0 | 0.07 | C5-WORSTCASE | **0.07** | High |
| C6 | Memory, context & configuration integrity | L2 | L2 | L1 | L0 | 0.35 | — | **0.35** | High |
| C7 | Third-party extensions | L1 | L1 | L2 | L3 | 0.40 | — | **0.40** | High |
| C8 | Secrets & sensitive-data protection | L3 | L3 | L3 | L1 | 0.65 | — | **0.65** | High |
| C9 | Audit & traceability | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C10 | Limits & kill switch | L3 | L2 | L2 | L2 | 0.57 | — | **0.57** | High |


ContextForge puts real access control in front of the tools it federates: every call is authenticated, checked against RBAC and team visibility, schema-validated, SSRF-filtered, rate-limited and timed out, and stored upstream credentials are encrypted and never shown to the model. It is still a relay rather than a firewall: tool results and upstream tool descriptions reach clients untouched, any user can register a new upstream server that is shared with everyone by default, and a client-supplied X-Upstream-Authorization header is always forwarded as the upstream credential. The audit trail and every content guardrail plugin ship switched off, so out of the box you cannot reconstruct who ran which tool with which arguments.

## Critical gaps
- Client-supplied X-Upstream-Authorization is always forwarded upstream as the Authorization header (MCP token passthrough); other upstream credential forwarding is not strictly scoped. (ASI03, T3, T9; C1) — [mcpgateway/utils/passthrough_headers.py:247-260](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/utils/passthrough_headers.py#L247-L260)
- An escape from the in-process Jinja2 template sandbox lands in the gateway process that holds the JWT secret, the credential encryption key and every stored upstream credential. (ASI05, T11; C4) — [mcpgateway/services/prompt_service.py:100](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/prompt_service.py#L100); [mcpgateway/config.py:679](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L679)
- Upstream tool output and descriptions are relayed with no provenance or filtering by default, so a hijacked client can read through one registered server and act or exfiltrate through another with shared stored credentials. (ASI01, T6, LLM01; C5) — [mcpgateway/services/tool_service.py:6105-6112](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/tool_service.py#L6105-L6112); [mcpgateway/plugins/__init__.py:41](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/plugins/__init__.py#L41)

## Criterion details

### C1 Identity & least privilege — 0.25 (high)

Every tool call on both the /rpc and /mcp paths is authenticated and checked against the caller's RBAC permission and team-scoped visibility before any upstream credential is attached, which is a genuine per-principal authorization layer. The credentials themselves are mostly static per upstream server and shared by every user who can see that server, and by default a newly registered server is public and every team role, including viewer, may execute tools. The gateway also always forwards a client-supplied X-Upstream-Authorization header as the upstream Authorization, which is MCP token passthrough and caps this criterion. A compromise exposes write access to every registered upstream system.

- **S L2:** Upstream calls use a static per-gateway stored credential (or per-user OAuth when configured), gated by RBAC and visibility before attachment; token exchange is opt-in. — [mcpgateway/services/tool_service.py:6105-6112](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/tool_service.py#L6105-L6112); [mcpgateway/services/tool_service.py:4940-4947](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/tool_service.py#L4940-L4947) (verified)
  - *To reach the next level:* Credentials are not narrowed per tool or per request; a single stored credential serves reads and writes for every permitted user.
- **C L3:** Both tool-call entry points enforce tools.execute RBAC plus Layer-1 visibility against the requesting principal. — [mcpgateway/transports/streamablehttp_transport.py:1997-2010](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/transports/streamablehttp_transport.py#L1997-L2010); [mcpgateway/main.py:11136](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/main.py#L11136); [mcpgateway/transports/streamablehttp_transport.py:1070](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/transports/streamablehttp_transport.py#L1070) (verified)
  - *To reach the next level:* RBAC is not enforced on every path, and the shared stored credential is used on behalf of any permitted user, so not all paths are fail-closed per principal.
- **D L2:** Auth is required by default, but personal-team owners get team_admin (gateways.create), viewers can execute tools, and new gateways default to public visibility. — [mcpgateway/config.py:366](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L366); [mcpgateway/schemas.py:3317](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/schemas.py#L3317); [mcpgateway/config.py:1102-1104](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L1102-L1104); [mcpgateway/config.py:1186](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L1186); [mcpgateway/bootstrap_db.py:485](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/bootstrap_db.py#L485) (verified)
  - *To reach the next level:* Default visibility for registered servers is public and tools.execute is granted to every team role, so least privilege needs operator hardening.
- **B L1:** The gateway holds long-lived write credentials for every registered upstream system (shared across users), so a hijacked identity reaches many systems. — [mcpgateway/services/tool_service.py:6105-6112](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/tool_service.py#L6105-L6112); [mcpgateway/config.py:679](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L679) (verified)
  - *To reach the next level:* No per-request, short-lived or read-only credential scoping for upstream systems.
- **Cap:** C1-PASSTHRU — The gateway unconditionally takes a client-supplied X-Upstream-Authorization header and forwards it to the downstream server.

### C2 Approval gates — 0.20 (high)

As a gateway, ContextForge leaves approval to the MCP host and only relays whatever readOnly/destructive hints the upstream server or the registering user supplied; it never derives them, so REST-wrapped tools that POST or DELETE usually carry no risk signal. It does offer a dry-run preview endpoint that validates arguments and resolves the target without calling it, but nothing forces a host to use it. Annotations can be set or changed by whoever registers or updates the tool, including the upstream server itself. Once a call is approved by the host, the gateway executes it against the upstream system with no undo.

- **S L1:** Annotations are stored and relayed but never derived or checked; a dry-run preview endpoint exists. — searched `rg -n -t py "readOnlyHint|destructiveHint"` in `mcpgateway` → 5 hits (All hits are schema field aliases/docstrings (schemas.py, common/models.py); nothing derives or enforces hints.); [mcpgateway/main.py:5988-5989](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/main.py#L5988-L5989) (verified)
  - *To reach the next level:* Not every mutating tool carries accurate readOnlyHint/destructiveHint; REST tools get none by default.
- **C L1:** Risk hints exist only on tools whose upstream or registrant provided them. — searched `rg -n -t py "readOnlyHint|destructiveHint"` in `mcpgateway` → 5 hits (All hits are schema field aliases/docstrings (schemas.py, common/models.py); nothing derives or enforces hints.) (verified)
  - *To reach the next level:* No server-side signal or confirmation covers every mutating tool.
- **D L1:** Hints come from the upstream server or the registering user and can be changed by them. — searched `rg -n -t py "readOnlyHint|destructiveHint"` in `mcpgateway` → 5 hits (All hits are schema field aliases/docstrings (schemas.py, common/models.py); nothing derives or enforces hints.) (verified)
  - *To reach the next level:* Hints are controlled by untrusted upstream servers rather than an operator policy.
- **B L0:** Federated tools reach arbitrary upstream write APIs with stored credentials and no rollback or quantity bounds beyond the generic rate limit. — [mcpgateway/services/tool_service.py:6105-6112](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/tool_service.py#L6105-L6112); [mcpgateway/config.py:3963](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L3963) (verified)
  - *To reach the next level:* No checkpoint, preview requirement or bounded quantities on consequential upstream actions.
- **Cap:** none
- **Notes:** The default-on LLM Chat feature (llmchat_enabled=True) embeds a LangGraph ReAct agent that calls gateway tools with no approval step; it needs an operator-configured LLM provider and llm.invoke permission, so it is footnoted rather than scored as the primary surface.

### C3 Tool & action scoping — 0.53 (high)

Every tool invocation, whatever the backend, is validated against the tool's JSON input schema in a killable worker before anything is sent, and REST tool URLs are re-validated against SSRF rules and pinned to the resolved IP after argument substitution, blocking cloud metadata, localhost and private ranges by default. Validation is only as tight as each tool's schema, and URL construction for upstream requests is not strictly constrained. The gateway ships no tools of its own, but anything registered is immediately callable by every user who can see it, and virtual servers are the only way to narrow the set.

- **S L2:** Typed JSON-schema validation on all calls and SSRF validation with DNS pinning on the final REST URL. — [mcpgateway/services/tool_service.py:5763-5764](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/tool_service.py#L5763-L5764); [mcpgateway/services/tool_service.py:6176-6184](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/tool_service.py#L6176-L6184); [mcpgateway/services/tool_service.py:6243](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/tool_service.py#L6243) (verified)
  - *To reach the next level:* URL construction and request handling for upstream calls are not strictly constrained.
- **C L3:** Schema validation runs in the shared resolver used by every integration type (REST, MCP, A2A, gRPC) and by preview. — [mcpgateway/services/tool_service.py:5763-5764](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/tool_service.py#L5763-L5764); [mcpgateway/services/tool_service.py:5762](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/tool_service.py#L5762) (verified)
  - *To reach the next level:* No central argument-level allowlist policy that every tool inherits beyond its own schema.
- **D L2:** No tools exist until registered, but every registered tool is enabled for all users with visibility; virtual servers allow per-endpoint tool subsets. — [mcpgateway/schemas.py:3317](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/schemas.py#L3317); [mcpgateway/config.py:780-783](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L780-L783) (verified)
  - *To reach the next level:* Registered write tools are enabled immediately with no read-only default set.
- **B L1:** Tools reach whatever upstream API the registrant configured with that API's stored credential; SSRF blocks internal networks. — [mcpgateway/config.py:780-783](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L780-L783); [mcpgateway/services/tool_service.py:6105-6112](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/tool_service.py#L6105-L6112) (verified)
  - *To reach the next level:* No per-tool quantity bounds or scoping beyond the upstream credential.
- **Cap:** none

### C4 Code-execution isolation — 0.33 (high)

The gateway never launches MCP servers or shell commands itself; the code it interprets is prompt templates (rendered with Jinja2's in-process SandboxedEnvironment) and jq filters on tool output, which run in forked worker processes with the environment cleared and a 2-second kill timer. Both are restriction layers inside the gateway's own user account rather than an OS boundary, and on non-Linux hosts or with one config switch jq runs in-process with no timeout. A template-sandbox escape lands inside the gateway process, which holds the JWT signing key, the credential encryption key and every stored upstream credential.

- **S L1:** Jinja2 SandboxedEnvironment (in-process filtering) and forked jq workers with a cleared environment, same OS user. — [mcpgateway/services/prompt_service.py:100](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/prompt_service.py#L100); [mcpgateway/utils/jq_runner.py:91](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/utils/jq_runner.py#L91); searched `rg -n "stdio_client|StdioServerParameters"` in `mcpgateway` → 0 hits (The gateway never launches MCP servers as subprocesses; stdio wrapping lives in the separate operator-run translate CLI.) (verified)
  - *To reach the next level:* No OS-level separation (dedicated user, hardened container) for template or filter execution.
- **C L2:** All prompt rendering uses the sandboxed environment and jq uses the worker pool on Linux. — [mcpgateway/services/prompt_service.py:100](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/prompt_service.py#L100); [mcpgateway/utils/jq_runner.py:218-219](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/utils/jq_runner.py#L218-L219) (verified)
  - *To reach the next level:* jq falls back to in-process execution on non-Linux platforms or when configured, so not every path is contained.
- **D L2:** Sandboxes are on by default; one setting (JQ_FILTER_EXECUTION=inprocess) removes the jq sandbox with only a log warning. — [mcpgateway/config.py:2775-2779](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2775-L2779); [mcpgateway/utils/jq_runner.py:218-219](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/utils/jq_runner.py#L218-L219) (verified)
  - *To reach the next level:* Disabling needs only a config value rather than an explicit, loudly named operator flag, and non-Linux silently degrades.
- **B L0:** An escape from the in-process Jinja sandbox runs in the gateway process holding the JWT secret, encryption secret and all upstream credentials, with full network egress. — [mcpgateway/config.py:679](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L679); [mcpgateway/services/prompt_service.py:100](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/prompt_service.py#L100) (verified)
  - *To reach the next level:* Template and filter execution are not isolated from the process that holds secrets and credentials.
- **Cap:** none

### C5 Untrusted input blast radius — 0.07 (high)

Tool results, upstream tool descriptions and A2A agent replies are relayed to clients exactly as received, with no provenance marking or untrusted-content flag the host could act on. The guardrail plugins that could filter content (deny lists, PII, moderation) are all disabled by default and are detection filters even when on. A client steered by malicious upstream content can use the same gateway session to read through one registered server and write or exfiltrate through another with stored credentials, and the gateway does nothing to stop that combination.

- **S L1:** Plain MCP content relayed from upstream with no provenance or untrusted flag. — searched `rg -n -i "untrusted|provenance"` in `mcpgateway/services/tool_service.py mcpgateway/transports/streamablehttp_transport.py` → 0 hits (No untrusted/provenance marking on relayed tool output or descriptions.) (verified)
  - *To reach the next level:* No structured separation of upstream-returned content from gateway metadata or provenance the host can act on.
- **C L0:** No source is distinguished: tool outputs, persisted upstream descriptions and A2A responses all enter with equal standing. — searched `rg -n -i "untrusted|provenance"` in `mcpgateway/services/tool_service.py mcpgateway/transports/streamablehttp_transport.py` → 0 hits (No untrusted/provenance marking on relayed tool output or descriptions.) (verified)
  - *To reach the next level:* No untrusted source is marked or limited.
- **D L0:** Content-filtering plugins exist but are disabled by default. — [mcpgateway/plugins/__init__.py:41](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/plugins/__init__.py#L41) (verified)
  - *To reach the next level:* No untrusted-content control is on by default.
- **B L0:** A hijacked client can read data through one upstream and act or exfiltrate through another, unattended, across shared multi-user credentials. — [mcpgateway/services/tool_service.py:6105-6112](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/tool_service.py#L6105-L6112); [mcpgateway/schemas.py:3317](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/schemas.py#L3317) (verified)
  - *To reach the next level:* No mode drops a Rule-of-Two leg by default.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.35 (high)

What persists is the shared catalog of tools, prompts and resources, including tool descriptions copied from upstream servers when they are registered, plus per-user LLM chat history in Redis that expires after an hour. Writes to the catalog require an authenticated user with create permission, and upstream changes are not pulled in automatically, but stored descriptions carry no provenance and newly registered servers default to public visibility, so one user's registration is served to every user's model. Configuration comes from operator files (.env in the working directory, plugins/config.yaml), not from untrusted workspaces.

- **S L2:** Catalog writes require RBAC create permissions and upstream refresh is manual by default, but persisted upstream descriptions are stored as-is. — [mcpgateway/config.py:2989](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2989); [mcpgateway/bootstrap_db.py:446](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/bootstrap_db.py#L446) (verified)
  - *To reach the next level:* No validation, approval or expiry on persisted upstream tool descriptions.
- **C L2:** The tool/prompt/resource catalog is RBAC-controlled; chat history is per-user with TTL. — [mcpgateway/config.py:2989](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2989); [mcpgateway/config.py:3374](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L3374) (verified)
  - *To reach the next level:* Upstream-sourced descriptions persisted at registration are not provenance-tagged.
- **D L1:** Visibility namespaces exist and are enforced in queries, but registered gateways default to public. — [mcpgateway/schemas.py:3317](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/schemas.py#L3317); [mcpgateway/config.py:1102-1104](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L1102-L1104) (verified)
  - *To reach the next level:* Per-team or per-user isolation is not the default for registered servers.
- **B L0:** A poisoned description persisted at registration is served to every user's model across sessions and can steer tool use. — [mcpgateway/schemas.py:3317](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/schemas.py#L3317); [mcpgateway/config.py:2989](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2989) (verified)
  - *To reach the next level:* Persisted catalog content reaches all users without review.
- **Cap:** none

### C7 Third-party extensions — 0.40 (high)

The extensions ContextForge loads are remote MCP servers and A2A agents that users register, plus optional Python plugins that are disabled by default. Remote servers run elsewhere and receive only their own configured credentials, so a malicious one cannot read the gateway's secrets, but nothing pins or verifies what a server serves, and any user who owns a personal team can register one that is public to everyone. Tool definitions are snapshotted at registration and only refreshed on request, which limits silent changes to the tool list but not to the server's behaviour.

- **S L1:** User-chosen remote servers, unpinned; definitions snapshotted at registration with manual refresh. — [mcpgateway/config.py:2989](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2989) (verified)
  - *To reach the next level:* No pinning, signature or allowlisted registry for upstream servers.
- **C L1:** The definition snapshot applies to federated MCP servers; A2A agents and plugins are not verified. — [mcpgateway/config.py:2989](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2989); [plugins/config.yaml:4-7](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/plugins/config.yaml#L4-L7) (verified)
  - *To reach the next level:* No verification for A2A agents or plugin code.
- **D L2:** Nothing is enabled by default and adding a server is an explicit API/UI action, but any personal-team owner can add one shared publicly. — [mcpgateway/plugins/__init__.py:41](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/plugins/__init__.py#L41); [mcpgateway/config.py:1186](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L1186); [mcpgateway/bootstrap_db.py:446](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/bootstrap_db.py#L446); [mcpgateway/schemas.py:3317](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/schemas.py#L3317) (verified)
  - *To reach the next level:* Registration is not restricted to admin scope and does not show permissions requested.
- **B L3:** Remote servers run off-host and receive only their own stored credentials (plus any client-forwarded upstream header). — [mcpgateway/services/tool_service.py:6105-6112](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/tool_service.py#L6105-L6112); [mcpgateway/utils/passthrough_headers.py:247-260](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/utils/passthrough_headers.py#L247-L260) (verified)
  - *To reach the next level:* Network and data access of an upstream server are not limited to what it declares.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.65 (high)

Upstream credentials are encrypted at rest with an Argon2id-derived Fernet key, attached server-side at call time and never placed in model context; logs default to ERROR level, payload logging and tracing are opt-in, and trace exports redact token, password and authorization fields. The weak points are the credentials themselves: they are long-lived, broadly scoped upstream keys, and the client-supplied X-Upstream-Authorization header is forwarded to the upstream server.

- **S L3:** Encrypted-at-rest credentials, SecretStr settings, masking in admin views and redaction in trace exports. — [mcpgateway/services/encryption_service.py:109-110](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/encryption_service.py#L109-L110); [mcpgateway/config.py:3265-3266](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L3265-L3266); [mcpgateway/admin.py:2681-2692](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/admin.py#L2681-L2692) (verified)
  - *To reach the next level:* No output scanning for secret patterns by default and upstream credentials are long-lived.
- **C L3:** Logs, traces, admin exports and the jq worker environment are covered; credentials never enter model-bound messages. — [mcpgateway/config.py:3265-3266](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L3265-L3266); [mcpgateway/utils/jq_runner.py:91](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/utils/jq_runner.py#L91) (verified)
  - *To reach the next level:* Upstream credential forwarding is not strictly scoped.
- **D L3:** Payload logging and observability are opt-in; default log level is ERROR. — [mcpgateway/config.py:2180](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2180); [mcpgateway/config.py:2229](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2229); [mcpgateway/config.py:2179](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2179) (verified)
  - *To reach the next level:* Stored transcripts and logs are not minimised or encrypted by default.
- **B L1:** Stored upstream keys are long-lived and broadly scoped; JWT session tokens expire in 20 minutes. — [mcpgateway/services/tool_service.py:6105-6112](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/tool_service.py#L6105-L6112); [mcpgateway/config.py:372](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L372) (verified)
  - *To reach the next level:* Upstream credentials are not short-lived or rotated automatically.
- **Cap:** none

### C9 Audit & traceability — 0.38 (high)

Out of the box the gateway records only per-tool metrics rows (tool, timestamp, latency, success) with no caller identity or arguments, buffered and flushed once a minute; the audit trail, security event log and permission audit are all off, and the default ERROR log level drops the INFO lines that name each invocation. After an incident you could see that a tool ran, but not who ran it or with what input, unless the operator had turned the audit features on.

- **S L1:** Default record is aggregate-friendly metrics without actor or arguments. — [mcpgateway/db.py:2597-2602](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/db.py#L2597-L2602); [mcpgateway/config.py:2179](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2179); [mcpgateway/services/structured_logger.py:354-357](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/structured_logger.py#L354-L357) (verified)
  - *To reach the next level:* No structured per-call record with arguments and actor by default.
- **C L2:** Metrics are recorded for every tool invocation regardless of backend. — [mcpgateway/db.py:2597-2602](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/db.py#L2597-L2602); [mcpgateway/config.py:2379](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2379); [mcpgateway/config.py:2389](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2389) (verified)
  - *To reach the next level:* Approvals, denials and configuration changes are not recorded by default.
- **D L2:** Metrics are on by default in the gateway's own database, outside any client's reach. — [mcpgateway/config.py:2379](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2379); [mcpgateway/config.py:2389](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2389) (verified)
  - *To reach the next level:* The gateway process can alter its own records and auditing is opt-in.
- **B L1:** Metrics are buffered and flushed every 60 seconds; failures are best-effort. — [mcpgateway/config.py:2462-2463](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2462-L2463) (verified)
  - *To reach the next level:* Records are not flushed per action and are lost on crash.
- **Cap:** none

### C10 Limits & kill switch — 0.57 (high)

Every tool call has a server-enforced timeout (60 seconds by default, overridable per tool), the MCP and tools endpoints are rate-limited per user at 100 requests a minute with lockout, and on the JSON-RPC path a cancellation request actually cancels the in-flight task. The advertised tool_rate_limit and tool_concurrent_limit settings are not enforced anywhere, there is no response-size cap on JSON tool results, and the embedded LLM chat agent sets no step limit of its own and registers a no-op cancel handler.

- **S L3:** Per-call timeouts on every invocation, Redis-backed per-user rate limiting, and task cancellation on /rpc. — [mcpgateway/config.py:2762-2765](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2762-L2765); [mcpgateway/services/tool_service.py:5821](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/tool_service.py#L5821); [mcpgateway/config.py:3949](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L3949); [mcpgateway/main.py:3446-3447](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/main.py#L3446-L3447); [mcpgateway/main.py:10684-10692](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/main.py#L10684-L10692) (verified)
  - *To reach the next level:* Cancellation does not reach every path (LLM chat registers a no-op), and concurrency limits are not enforced.
- **C L2:** Limits cover all tool calls; the embedded chat agent loop has no step cap of its own. — [mcpgateway/services/mcp_client_chat_service.py:2823](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/mcp_client_chat_service.py#L2823); searched `rg -n "recursion_limit"` in `mcpgateway` → 0 hits (The embedded LLM-chat agent loop never sets a step limit; it inherits LangGraph's library default.); [mcpgateway/services/mcp_client_chat_service.py:2464](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/services/mcp_client_chat_service.py#L2464) (verified)
  - *To reach the next level:* Sub-agent (LLM chat) loops and background work do not share a bounded budget.
- **D L2:** Sensible defaults, operator-configurable; per-tool timeouts can be raised by tool registrants. — searched `rg -n "settings\.tool_rate_limit|settings\.tool_concurrent_limit"` in `mcpgateway` → 2 hits (Both hits are admin.py display of the values; neither setting is enforced anywhere.); [mcpgateway/config.py:2762-2765](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L2762-L2765) (verified)
  - *To reach the next level:* No hard ceilings; tool_concurrent_limit/tool_rate_limit are configured but never enforced.
- **B L2:** 60-second calls and 100 rpm per user bound runaway use, and /rpc cancel stops in-flight work. — [mcpgateway/config.py:3963](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/config.py#L3963); [mcpgateway/main.py:10684-10692](https://github.com/IBM/mcp-context-forge/blob/9bb9c2ac58af4cd126c05e8153f9025770646bd2/mcpgateway/main.py#L10684-L10692) (verified)
  - *To reach the next level:* No total cost or wall-clock ceiling per client session.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Upstream tool results and persisted upstream tool descriptions relayed verbatim (mcpgateway/services/tool_service.py:5762) · [B] sensitive data/systems: Shared stored upstream credentials attached per call (mcpgateway/services/tool_service.py:6105) · [C] state change / egress: Any registered REST/MCP/A2A write tool reachable in the same session (mcpgateway/services/tool_service.py:6243) · Same default session? Yes

## Highest-impact improvements
1. Turn on audit_trail and record every tool invocation with caller, arguments and result, flushed per call. — C9 S L1→L2, +0.075 before caps (Playbook 1 step 3)
2. Gate X-Upstream-Authorization and other client-credential passthrough behind an explicit per-gateway opt-in (lifts the passthrough cap) and default new gateways to team visibility. — C1 D L2→L3, +0.050 before caps (Playbook 4)
3. Default new gateways to private/team visibility and drop tools.execute from the viewer role. — C6 D L1→L2, +0.050 before caps (Playbook 2)
4. Derive destructiveHint/readOnlyHint for REST tools from the HTTP method and refuse unannotated mutating tools in a read-only server mode. — C2 S L1→L2, +0.075 before caps (Playbook 5)
5. Harden URL construction and request handling for upstream calls. — C3 S L2→L3, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Version is null: the shallow clone carries no tag at HEAD (pyproject.toml declares 1.0.11).
- The experimental Rust MCP runtime, Helm chart, docker-compose stacks, SSO providers and the 40+ optional plugins were not examined in depth; only their default enablement was checked.
- The repo's AGENTS.md/CLAUDE.md were read as data; no text attempting to steer reviewers was found.
