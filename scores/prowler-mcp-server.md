# Defense-in-Depth Score: Prowler MCP Server

**Repo:** https://github.com/prowler-cloud/prowler (`mcp_server`) · **Commit:** `383a9bf9032c503c6e920d9f94f7712d046d3a7b` (0.9.0) · **Reviewed:** 2026-10-03
**What it is:** Cloud security posture platform; ships the Prowler MCP Server and Lighthouse AI assistant
**Category:** Cybersecurity
**Scored configuration:** Local stdio mode with a user-supplied PROWLER_API_KEY (README mode 2) and all tools registered; the HTTP mode that forwards the caller's bearer token is footnoted.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 4.7 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L1 | L2 | 0.23 | C1-SELFESC | **0.23** | High |
| C2 | Approval gates | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C3 | Tool & action scoping | L2 | L2 | L0 | L2 | 0.40 | — | **0.40** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L0 | 0.07 | C5-WORSTCASE | **0.07** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L0 | L1 | L1 | L1 | 0.17 | C8-MODELSECRETS | **0.17** | High |
| C9 | Audit & traceability | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |

Controls where a risk surface exists: 1.73 / 7.0 (25%); 3 criteria scored SA (surface absent).

The server is a thin, well-typed wrapper over the Prowler API with careful error masking, but it ships every read and write tool at once, with no risk annotations, no read-only mode, no confirmation or dry-run, and almost no record of what was called. A hijacked model can delete providers, replace the mutelist (hiding findings), change user roles, and create export integrations that send scan data to a destination it names. The cloud credentials used to onboard providers also pass through the model as plain tool arguments. Safety rests almost entirely on the host's approval step and the permissions of the Prowler API key.

## Critical gaps
- set_user_role can reassign any user's role, including the role of the key the server runs as, with no restriction or logging in the server. (ASI03, T3; C1) — [mcp_server/prowler_mcp_server/prowler_app/tools/roles.py:128](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/roles.py#L128); [mcp_server/prowler_mcp_server/prowler_app/tools/roles.py:190](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/roles.py#L190)
- A hijacked model could combine tenant data reads with deletions, mutelist replacement and export integrations to an arbitrary bucket, with no server-side approval or egress control. (ASI01, LLM01; C5) — [mcp_server/prowler_mcp_server/prowler_app/tools/integrations.py:161](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/integrations.py#L161); [mcp_server/prowler_mcp_server/prowler_app/tools/providers.py:296](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/providers.py#L296); [mcp_server/prowler_mcp_server/prowler_app/tools/muting.py:183](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/muting.py#L183)
- Cloud and integration secrets are supplied as model-written tool arguments, so they pass through the model and the host's transcripts. (ASI03, LLM02; C8) — [mcp_server/prowler_mcp_server/prowler_app/tools/providers.py:145](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/providers.py#L145); [mcp_server/prowler_mcp_server/prowler_app/tools/providers.py:168](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/providers.py#L168)

## Criterion details

### C1 Identity & least privilege — 0.23 (high)

The server holds no identity of its own: in stdio mode it uses one Prowler API key from the environment, and in HTTP mode it forwards whatever bearer token or API key the caller sends to the Prowler API. Every tool, read or write, uses that same credential, so authority is whatever role the key has and the server does nothing to narrow it. One tool, set_user_role, lets the agent change any user's role (including the key owner's) with no check beyond the API's own permissions. The Prowler API's own role checks are the only remaining layer.

- **S L0:** One credential serves every tool and the server can assign roles to any user, including the one it runs as. — [mcp_server/prowler_mcp_server/prowler_app/tools/roles.py:128](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/roles.py#L128); [mcp_server/prowler_mcp_server/prowler_app/tools/roles.py:190](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/roles.py#L190) (verified)
  - *To reach the next level:* No per-tool or read-only credential and no restriction on changing role assignments.
- **C L1:** All tools go through one shared client, but it applies the single ambient credential to everything and checks nothing per tool. — [mcp_server/prowler_mcp_server/prowler_app/utils/api_client.py:89](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/utils/api_client.py#L89) (verified)
  - *To reach the next level:* No per-tool authorization layer in code.
- **D L1:** Every read and write tool is registered by default and the key's role is chosen entirely by the operator; there is no read-only mode. — [mcp_server/prowler_mcp_server/prowler_app/tools/base.py:96](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/base.py#L96); searched `rg -n -i 'read_only_mode|readonly_mode|PROWLER_MCP_READ' --glob '*.py'` in `mcp_server/prowler_mcp_server` → 0 hits (no read-only switch exists in the server code) (verified)
  - *To reach the next level:* No read-only default or elevation step for write tools.
- **B L2:** Write access to one system (the Prowler tenant: providers, stored cloud credentials, roles, mutelist, integrations); the Prowler API's own RBAC bounds what a given key can do. — [mcp_server/prowler_mcp_server/prowler_app/tools/roles.py:190](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/roles.py#L190); [mcp_server/prowler_mcp_server/prowler_app/tools/providers.py:323](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/providers.py#L323) (verified)
  - *To reach the next level:* Credentials are long-lived keys that are not narrowed to reads or to a single provider.
- **Cap:** C1-SELFESC — set_user_role lets the agent change any user's role, including that of the key it runs as, with no server-side restriction.
- **Notes:** HTTP mode reads the Authorization header per request (prowler_app/utils/auth.py) and only checks for a JWT's exp claim without verifying a signature; validation is left to the Prowler API.

### C2 Approval gates — 0.05 (high)

No tool carries any risk annotation (read-only, destructive, idempotent hints), so a host cannot tell the read tools from tools that delete providers or replace the mutelist. There is no dry-run, preview, confirmation step or read-only mode. Destructive tools describe their danger only in docstrings, which a host cannot enforce. Deletions of providers (with their scans and findings), the mutelist and Jira work items cannot be undone.

- **S L0:** Tools are registered with a bare mcp.tool(method) and no annotations, so nothing machine-readable separates reads from destructive writes. — [mcp_server/prowler_mcp_server/prowler_app/tools/base.py:96](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/base.py#L96); searched `rg -n 'readOnlyHint|destructiveHint|annotations' --glob '*.py'` in `mcp_server/prowler_mcp_server` → 0 hits (no annotation is set on any tool) (verified)
  - *To reach the next level:* Accurate readOnlyHint/destructiveHint on every tool would be the first step.
- **C L0:** Nothing in the server gates any tool, including the most consequential ones such as delete_provider and set_user_role. — [mcp_server/prowler_mcp_server/prowler_app/tools/providers.py:296](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/providers.py#L296); [mcp_server/prowler_mcp_server/prowler_app/tools/roles.py:128](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/roles.py#L128) (verified)
  - *To reach the next level:* A server-side confirmation or gate that every write tool must pass.
- **D L0:** No approval or confirmation feature exists in the server, so nothing is on by default. — searched `rg -n -i 'elicit|confirm_|dry_run|require_approval' --glob '*.py'` in `mcp_server/prowler_mcp_server` → 0 hits (no elicitation, confirmation or dry-run code) (verified)
  - *To reach the next level:* Any confirmation step enabled by default.
- **B L1:** Mostly irreversible: provider deletion removes scans, findings and resources, the mutelist deletion cannot be undone, and Jira work items cannot be removed; mute rules and scans are the reversible cases. — [mcp_server/prowler_mcp_server/prowler_app/tools/providers.py:304](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/providers.py#L304); [mcp_server/prowler_mcp_server/prowler_app/tools/integrations.py:685](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/integrations.py#L685) (verified)
  - *To reach the next level:* No previews, undo, rate limits or quantity bounds on consequential actions.
- **Cap:** none
- **Notes:** The host owns the approval prompt; this scores only what the server gives it. Docstrings warn about destructive tools, but prompts are not controls.

### C3 Tool & action scoping — 0.40 (high)

Arguments are typed and validated in several places: non-blank identifiers, page sizes capped at 1000, date formats and ranges, enumerations, and an HTTPS plus single-host allowlist for the one external fetch. The Prowler Hub client encodes path segments safely. Most app tools, however, join model-supplied IDs into API paths with plain f-strings, and credential and mutelist arguments are free-form dictionaries. Every tool, read and write, is enabled with no way to disable groups.

- **S L2:** Pydantic types, NonBlankStr, bounded page size, date checks and a host allowlist exist, but IDs in app tools are interpolated into API paths without encoding. — [mcp_server/prowler_mcp_server/lib/types.py:18](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/lib/types.py#L18); [mcp_server/prowler_mcp_server/prowler_app/utils/api_client.py:402](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/utils/api_client.py#L402); [mcp_server/prowler_mcp_server/prowler_app/tools/roles.py:97](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/roles.py#L97) (verified)
  - *To reach the next level:* Encode every path segment (as lib/urls.py does for Hub) and validate IDs as UUIDs.
- **C L2:** Most tools validate through the shared client and NonBlankStr, but path encoding is only used in the Hub server and one finding-groups helper, not in the app tools. — [mcp_server/prowler_mcp_server/lib/urls.py:21](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/lib/urls.py#L21); [mcp_server/prowler_mcp_server/prowler_app/tools/finding_groups.py:63](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/finding_groups.py#L63) (verified)
  - *To reach the next level:* One central validation layer that every tool and new tool inherits.
- **D L0:** Every tool, including write and delete tools, is registered at startup with no setting to select groups or read-only sets. — [mcp_server/prowler_mcp_server/prowler_app/tools/base.py:96](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/base.py#L96); [mcp_server/prowler_mcp_server/prowler_app/utils/tool_loader.py:84](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/utils/tool_loader.py#L84) (verified)
  - *To reach the next level:* A read-only default tool set with writes needing explicit enabling.
- **B L2:** Scoped to one Prowler tenant with full write inside it; page size and date windows bound reads, but finding_ids and mutelist/credential dictionaries are unbounded. — [mcp_server/prowler_mcp_server/prowler_app/tools/integrations.py:676](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/integrations.py#L676) (verified)
  - *To reach the next level:* Quantity bounds on write inputs and narrower tools for the highest-impact writes.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

The server contains no code path that runs model-supplied text as code: no shell, subprocess, eval, exec or deserialization of untrusted data. Attack-path queries are chosen by ID and run by the Prowler API, not interpreted here. The only dynamic import loads tool modules from the server's own package.

- **Structural absence:** searched `rg -n -S 'subprocess|os\.system|os\.popen|\beval\(|\bexec\(|pickle|shell=True|compile\('` in `mcp_server/prowler_mcp_server` → 0 hits (no execution primitives anywhere in the scored subpath); searched `rg -n -S 'importlib\.import_module|__import__'` in `mcp_server/prowler_mcp_server` → 2 hits (both hits are tool_loader.py importing the server's own fixed tools package, not model-reachable)
- **Notes:** Attack-path queries execute server-side in the Prowler API (not reviewed).

### C5 Untrusted input blast radius — 0.07 (high)

Tools return tenant data an outsider can influence: finding text, resource names and tags, scan and documentation excerpts, all passed through as plain structured fields with no provenance or untrusted marker. The server cannot see what the host does with it and offers no mode that removes the write or egress tools after such content is read. A hijacked model could therefore chain reads of attacker-influenced data into deletions, mutelist replacement, or an export integration pointing at a destination it names, with no server-side gate.

- **S L1:** Outputs are structured model dumps with no provenance or untrusted flag, and documentation excerpts are returned as raw text. — [mcp_server/prowler_mcp_server/prowler_documentation/search_engine.py:117](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_documentation/search_engine.py#L117) (verified)
  - *To reach the next level:* Provenance and an untrusted marker on returned content, and a read-only mode.
- **C L0:** No input source is distinguished: findings, resources, docs and Hub content all return with the same standing as any other tool result. — [mcp_server/prowler_mcp_server/prowler_app/tools/findings.py:29](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/findings.py#L29) (verified)
  - *To reach the next level:* Per-source handling for every returned source.
- **D L0:** There is no untrusted-input control to be on by default. — searched `rg -n -i 'untrusted|sanitiz' --glob '*.py'` in `mcp_server/prowler_mcp_server` → 0 hits (no handling of untrusted content in the server) (verified)
  - *To reach the next level:* Any default-on control.
- **B L0:** Data leak plus irreversible action are both available unattended from the server: export integrations send scan outputs to a named destination and delete tools cannot be undone, with no server-side approval. — [mcp_server/prowler_mcp_server/prowler_app/tools/integrations.py:161](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/integrations.py#L161); [mcp_server/prowler_mcp_server/prowler_app/tools/providers.py:296](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/providers.py#L296) (verified)
  - *To reach the next level:* Server-side gates or a mode that drops the egress or write legs.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 1.00 (high)

The server has no memory, retrieval store or auto-loaded configuration: it reads no workspace files, no dotenv file and no instruction files, and keeps no state between stateless HTTP requests. Settings come from environment variables and command-line flags only. Persisted Prowler mutelist and mute rules are stored in the tenant, and are scored as state-changing actions elsewhere.

- **Structural absence:** searched `rg -n -S 'load_dotenv|dotenv|memory|vector|embedding|AGENTS\.md|CLAUDE\.md|remember'` in `mcp_server/prowler_mcp_server` → 1 hits (single hit is the word 'vectors' in an attack-paths docstring); [mcp_server/prowler_mcp_server/main.py:51](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/main.py#L51)
- **Notes:** Tenant-side mutelist and mute rules persist model-written configuration that changes future findings; treated under C2/C3/C5.

### C7 Third-party extensions — 1.00 (high)

The server loads no third-party code at runtime: tools are discovered from its own package, it installs nothing, and the external content it fetches (Prowler Hub JSON, docs pages, check source from raw.githubusercontent.com) is returned as text and never executed. Dependencies are pinned in the lockfile and the container build uses a frozen sync with a digest-pinned base image.

- **Structural absence:** searched `rg -n -S 'pip install|npx|entry_points|plugin'` in `mcp_server/prowler_mcp_server` → 0 hits (no installer, plugin or entry-point loading); [mcp_server/prowler_mcp_server/prowler_app/utils/tool_loader.py:84](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/utils/tool_loader.py#L84)

### C8 Secrets & sensitive-data protection — 0.17 (high)

The server's own key comes from an environment variable and is not logged. Error handling is careful: upstream error bodies are logged but not relayed to the model, and unhandled failures are masked. However, onboarding cloud providers and integrations requires the model to write the cloud secrets (AWS keys, Azure client secrets, GCP private keys, kubeconfigs, Jira tokens) as tool arguments, so they travel through the model and the host's transcripts. There is no redaction of logged upstream bodies, and the stored secrets are long-lived.

- **S L0:** Secret material is placed into model context by design through the credentials argument of connect_provider and the integration creators. — [mcp_server/prowler_mcp_server/prowler_app/tools/providers.py:145](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/providers.py#L145); [mcp_server/prowler_mcp_server/prowler_app/tools/providers.py:168](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/providers.py#L168) (verified)
  - *To reach the next level:* Opaque credential handles or an out-of-band channel so secrets never pass through the model.
- **C L1:** Error text is masked for the model on all sub-servers, but logs, tool arguments and model-bound messages carry secrets unredacted. — [mcp_server/prowler_mcp_server/server.py:11](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/server.py#L11); [mcp_server/prowler_mcp_server/prowler_app/utils/api_client.py:109](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/utils/api_client.py#L109) (verified)
  - *To reach the next level:* Redaction on logs, errors and model-bound messages on all major paths.
- **D L1:** No telemetry is sent and default logging is at info level without auth headers, but there is no redaction layer to leave on. — searched `rg -n -i 'sentry|posthog|telemetry|redact' --glob '*.py'` in `mcp_server/prowler_mcp_server` → 0 hits (no telemetry or redaction code) (verified)
  - *To reach the next level:* Redaction always on and content-free logs by default.
- **B L1:** Long-lived cloud and integration credentials (moderately scoped by whatever the operator supplies) reach the model through tool arguments. — [mcp_server/prowler_mcp_server/prowler_app/tools/providers.py:168](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/providers.py#L168) (verified)
  - *To reach the next level:* Short-lived or scoped credentials and opaque handles.
- **Cap:** C8-MODELSECRETS — The documented provider and integration onboarding path requires the user's cloud secrets to be sent as model-written tool arguments.
- **Notes:** The credentials argument is optional, so providers can be created without secrets; the cap reflects the only supported way to attach credentials through the server.

### C9 Audit & traceability — 0.30 (high)

There is no per-call record of tool use. Some write tools (providers, integrations, mutelist, scan trigger) log an info line to the server's standard log, and failures are logged by a shared middleware, but roles, users, findings and resources tools log nothing, including set_user_role. Logs carry no caller identity or arguments, and nothing is tamper-evident or exported. The process log is outside the model's control.

- **S L1:** Unstructured info lines on some write tools and a middleware warning on failures; no structured per-call record with arguments, status and time. — [mcp_server/prowler_mcp_server/lib/errors.py:323](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/lib/errors.py#L323); [mcp_server/prowler_mcp_server/prowler_app/tools/scans.py:215](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/scans.py#L215) (verified)
  - *To reach the next level:* A structured record of every tool call with arguments and result status.
- **C L1:** Only some tools log; the role-change tool and all read tools have no logging at all. — searched `rg -n 'logger' --glob '*.py'` in `mcp_server/prowler_mcp_server/prowler_app/tools/roles.py` → 0 hits (set_user_role and the other role tools never log) (verified)
  - *To reach the next level:* Logging on all built-in tools.
- **D L2:** Logging is on by default and written by the server process to its own log stream, outside anything the model can write, but without an operator-level switch or integrity protection. — [mcp_server/prowler_mcp_server/lib/logger.py:4](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/lib/logger.py#L4) (verified)
  - *To reach the next level:* Records written by a separate component that cannot be disabled without a logged change.
- **B L1:** Best-effort logging through the library logger; actions proceed regardless of whether a record is written. — [mcp_server/prowler_mcp_server/lib/errors.py:315](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/lib/errors.py#L315) (verified)
  - *To reach the next level:* Durable per-action records and fail-closed logging for high-risk actions.
- **Cap:** none

### C10 Limits & kill switch — 0.50 (high)

The server enforces some bounds on its own work: page size capped at 1000 (50 for events, 20 for docs search), 30 second HTTP timeouts, 60 second and Jira dispatch polling timeouts, and a two-day window on historical finding queries. It has no rate limits, concurrency limits, size cap on write inputs such as finding_ids, or explicit cancellation of in-flight work, and callers choose how close to the ceiling to run.

- **S L2:** Server-enforced caps on several operations (page size, date window, timeouts) but not on every operation and no rate or concurrency limit. — [mcp_server/prowler_mcp_server/prowler_app/utils/api_client.py:402](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/utils/api_client.py#L402); [mcp_server/prowler_mcp_server/prowler_app/utils/api_client.py:63](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/utils/api_client.py#L63) (verified)
  - *To reach the next level:* Caps on every operation plus rate or concurrency limits.
- **C L2:** Per-request and polling timeouts exist on the shared client and long-running tools, but there is no shared budget across calls. — [mcp_server/prowler_mcp_server/prowler_app/utils/api_client.py:309](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/utils/api_client.py#L309) (verified)
  - *To reach the next level:* Caps that apply across background tasks and concurrent calls.
- **D L2:** Sensible fixed defaults in code, with the model choosing page sizes within a ceiling. — [mcp_server/prowler_mcp_server/prowler_app/tools/resources.py:363](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/tools/resources.py#L363) (verified)
  - *To reach the next level:* Hard ceilings the model cannot raise within the ceiling and rate limits.
- **B L2:** Moderate ceilings; a runaway caller is bounded per request, but nothing limits the number of calls and a timed-out poll leaves the task running upstream. — [mcp_server/prowler_mcp_server/prowler_app/utils/api_client.py:343](https://github.com/prowler-cloud/prowler/blob/383a9bf9032c503c6e920d9f94f7712d046d3a7b/mcp_server/prowler_mcp_server/prowler_app/utils/api_client.py#L343) (verified)
  - *To reach the next level:* Tight per-run ceilings and cancellation of pending work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: mcp_server/prowler_mcp_server/prowler_documentation/search_engine.py:117 and findings/resources returned verbatim from the tenant (attacker-nameable cloud resource data) · [B] sensitive data/systems: mcp_server/prowler_mcp_server/prowler_app/tools/providers.py:145 cloud credentials as tool args; findings and resources of the whole tenant · [C] state change / egress: mcp_server/prowler_mcp_server/prowler_app/tools/integrations.py:161 S3 export integration; mcp_server/prowler_mcp_server/prowler_app/tools/providers.py:296 delete_provider · Same default session? Yes

## Highest-impact improvements
1. Set readOnlyHint, destructiveHint and idempotentHint on every tool so hosts can gate deletions, role changes and mutelist writes. — C2 S L0→L2, +0.150 before caps (Playbook 5)
2. Add a server-enforced read-only mode (env flag) that registers only read tools, defaulting to read-only. — C3 D L0→L3, +0.150 before caps (Playbook 3)
3. Stop accepting cloud secrets as tool arguments; use a pre-stored credential reference or out-of-band entry. — C8 S L0→L3, +0.225 before caps (Playbook 4)
4. Log every tool call (tool, caller identity, arguments without secrets, outcome) in structured form. — C9 S L1→L3, +0.150 before caps (Playbook 1 step 3)
5. Block set_user_role on the caller's own user and encode every ID with url_path in app tools. — C1 S L0→L2, +0.150 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static review of the mcp_server subpath at the pinned commit only; nothing was installed, built, run, or network-probed, and the core Prowler scanner, the Prowler API, and Lighthouse AI were not reviewed.
- The Prowler API enforces its own RBAC server-side and was not reviewed; ratings credit it only as an independent layer in B where the MCP code itself relies on it, and never as an MCP control.
- FastMCP 3.4.5 behaviour (tool annotations default to none, error masking, stateless HTTP) is inferred from the library's documented behaviour, not from its source.
- C6 and C7 are marked structurally absent for agent memory and runtime third-party code; the Prowler mutelist, mute rules and integrations are persisted in the Prowler tenant and are scored as state-changing actions under C2, C3 and C5 rather than as agent memory.
- Repository text (AGENTS.md, README, docstrings) was treated as data; no instructions aimed at reviewers were found.
