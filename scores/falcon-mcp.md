# Defense-in-Depth Score: Falcon MCP

**Repo:** https://github.com/CrowdStrike/falcon-mcp · **Commit:** `9bc0efeb92e38ca0667e5a18847a7cbf09dcd206` (0.19.0) · **Reviewed:** 2026-10-03
**What it is:** Connects AI agents to CrowdStrike Falcon for security analysis and threat hunting
**Category:** Cybersecurity
**Scored configuration:** Local stdio transport, all modules and all tools registered, one operator-supplied Falcon API client, read-only mode and API key off.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 4.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C2 | Approval gates | L4 | L4 | L0 | L3 | 0.75 | G1 | **0.50** (alt) | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L1 | 0.12 | G1 | **0.12** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L1 | 0.30 | C6-REPOCONFIG | **0.25** | Low |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L2 | 0.35 | — | **0.35** | High |
| C9 | Audit & traceability | L0 | L1 | L0 | L0 | 0.07 | — | **0.07** | High |
| C10 | Limits & kill switch | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |

Controls where a risk surface exists: 2.68 / 8.0 (34%); 2 criteria scored SA (surface absent).

Falcon MCP gives an AI host broad access to a CrowdStrike tenant: it can search hosts, detections and logs and, by default, also change policies, IOCs, host groups and exclusions, run read-only RTR commands on endpoints, and execute workflows. The server's own safeguards are accurate risk labels on tools and an opt-in read-only mode; it has no approval gate, no audit log of tool calls, and no marking of untrusted content. Run it with --read-only, a least-privilege API client, and a host that asks for approval on state-changing tools.

## Critical gaps
- A .env file auto-loaded at startup can silently set the Falcon credentials, API base URL and the server's read-only and tool-selection switches. (ASI06, T1; C6) — [falcon_mcp/server.py:682](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/server.py#L682); [falcon_mcp/client.py:53-55](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/client.py#L53-L55)

## Criterion details

### C1 Identity & least privilege — 0.50 (high)

The server authenticates to the CrowdStrike Falcon API with a single operator-supplied API client (client ID and secret from environment variables) and uses that one client for every tool, read or write. The server does not request, narrow or check scopes itself; what the credential can do is whatever the operator granted the API client, and Falcon enforces that on its side. There is no per-request or per-user authorization, and the optional HTTP API key is one shared secret for all callers. If the credential is hijacked the damage is bounded by the API client's scopes, which can include host, policy, IOC and RTR write access across the tenant (and child tenants when a member CID is set).

- **S L2:** One static API client credential is used for all reads and writes; the server never narrows or requests scopes. — [falcon_mcp/client.py:51-52](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/client.py#L51-L52); [falcon_mcp/client.py:87](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/client.py#L87) (verified)
  - *To reach the next level:* No per-tool read-only versus write credentials and no short-lived or downscoped tokens.
- **C L2:** Every built-in tool goes through the same shared client; no tool builds its own privileged client, but no per-principal authorization exists. — [falcon_mcp/modules/base.py:82-90](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/base.py#L82-L90); [falcon_mcp/common/auth.py:90](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/common/auth.py#L90) (verified)
  - *To reach the next level:* Authorization is not evaluated per requesting principal; the HTTP transport has only one shared key.
- **D L2:** Default role is whatever the operator's API client holds; all modules and all write tools are registered by default and widening is a plain config change. — [falcon_mcp/server.py:127](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/server.py#L127); [falcon_mcp/server.py:73](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/server.py#L73) (verified)
  - *To reach the next level:* Default is not read-only; write tools need no explicit elevation.
- **B L2:** Credential failure exposes read of many sensitive Falcon data sets and write to one vendor system (hosts, policies, IOCs, RTR), limited by the API client's scopes, which Falcon enforces independently of this server. — [falcon_mcp/client.py:60](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/client.py#L60); [falcon_mcp/common/api_scopes.py:13](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/common/api_scopes.py#L13) (verified)
  - *To reach the next level:* Not read-only or narrowly scoped writes; a member CID widens reach to child tenants.
- **Cap:** none

### C2 Approval gates — 0.50 (high)

The server does not run an approval gate; the host decides. What it offers the host is accurate risk annotations: mutating tools are separate from read tools and carry readOnlyHint=false, with destructiveHint=true on deletes. Unannotated tools default to a read-only label, which fails open for any future tool whose author forgets an annotation. Only the quarantine module has a preview tool for destructive actions. A server-enforced read-only mode exists, withholds anything not explicitly read-only, and wins over the allow-list, but it is off by default. In the default configuration every write, delete and workflow-execute tool is registered and no rate limit or quantity bound applies to bulk deletes.

- **default configuration** (default; raw 0.45 → 0.45)
  - **S L2:** Read and write tools are separate and mutating tools carry readOnlyHint=false and destructiveHint on deletes, with the read-only label as the fallback for any tool lacking an annotation. — [falcon_mcp/modules/base.py:126](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/base.py#L126); [falcon_mcp/modules/host_groups.py:75](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/host_groups.py#L75); [falcon_mcp/modules/quarantine.py:39-42](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/quarantine.py#L39-L42) (verified)
    - *To reach the next level:* Only one preview tool exists for destructive actions, and the default fallback label is read-only rather than unknown.
  - **C L2:** All mutating tools I checked carry annotations, but the checks are by operation scan and the dynamic-mode executor registers with no annotations; unannotated tools default to read-only. — [falcon_mcp/dynamic.py:514-515](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/dynamic.py#L514-L515); [falcon_mcp/modules/rtr.py:66-76](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/rtr.py#L66-L76) (verified)
    - *To reach the next level:* Not every destructive tool has a preview or confirmation step the host must complete.
  - **D L2:** Annotations are always on and cannot be switched off by the model; the server-side enforcement mode is opt-in. — [falcon_mcp/server.py:651](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/server.py#L651) (verified)
    - *To reach the next level:* Read-only enforcement is not on by default.
  - **B L1:** Deletes of host groups, IOCs, policies, exclusions, quarantined files and RTR sessions and workflow execution are irreversible or hard to undo, with no rate limits or quantity bounds on bulk calls. — [falcon_mcp/modules/fusion.py:101-110](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/fusion.py#L101-L110); [falcon_mcp/modules/quarantine.py:304](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/quarantine.py#L304) (verified)
    - *To reach the next level:* Previews exist only for quarantine actions; no undo and no rate limits on consequential actions.
- **opt-in --read-only mode** (alt; raw 0.75, cap G1 → 0.50) ← counted
  - **S L4:** Read-only mode removes every tool whose annotations are not readOnlyHint=True, even if the allow-list names it, and treats unclassified tools as mutating. — [falcon_mcp/tool_filter.py:158-168](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/tool_filter.py#L158-L168) (verified)
    - *To reach the next level:* None for this mechanism.
  - **C L4:** The filter is applied to all registered tools, and to the dynamic-mode catalog, with unknown tools withheld rather than exposed. — [falcon_mcp/server.py:346-361](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/server.py#L346-L361) (verified)
    - *To reach the next level:* None.
  - **D L0:** Read-only mode is opt-in and defaults to false. — [falcon_mcp/server.py:73](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/server.py#L73) (verified)
    - *To reach the next level:* Would need to be on by default.
  - **B L3:** With the mode on, no tenant-state-changing tool is registered; only low-impact search-job creation remains. — [falcon_mcp/tool_filter.py:153-154](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/tool_filter.py#L153-L154); [falcon_mcp/modules/ngsiem.py:142-146](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/ngsiem.py#L142-L146) (verified)
    - *To reach the next level:* Relies on annotation accuracy for every tool.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C3 Tool & action scoping — 0.45 (high)

Tools are typed with Pydantic schemas, with numeric bounds on limits and required-ID checks on delete tools, and some inputs are sanitised (NGSIEM repository names reject path separators, identity-investigation values are JSON-encoded). Filters are passed through to the Falcon API as free-form FQL, CQL queries pass through unvalidated, and the RTR tools accept a free-form base command and command string, with the read-only restriction left to the API endpoint rather than checked in this server. By default all modules and all write and delete tools are enabled, though modules, individual tools and a read-only flag can be selected by the operator.

- **S L2:** Typed schemas with bounds on limits and ID presence checks, and a path-character check for the NGSIEM repository name, but filters, CQL and RTR commands are passed through without allowlists. — [falcon_mcp/modules/ngsiem.py:88](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/ngsiem.py#L88); [falcon_mcp/modules/rtr.py:509-515](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/rtr.py#L509-L515); [falcon_mcp/modules/hosts.py:111-115](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/hosts.py#L111-L115) (verified)
  - *To reach the next level:* No allowlist for RTR base commands or for filter and query content.
- **C L2:** Most built-in tools use typed fields with bounds, but validation is uneven (the RTR command string and query strings are unchecked). — [falcon_mcp/modules/rtr.py:534-543](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/rtr.py#L534-L543); [falcon_mcp/modules/base.py:240-278](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/base.py#L240-L278) (verified)
  - *To reach the next level:* No shared validation layer that new tools inherit.
- **D L2:** Modules, individual tools and read-only are selectable, but the default set includes write and delete tools. — [falcon_mcp/server.py:124-127](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/server.py#L124-L127); [falcon_mcp/server.py:664-672](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/server.py#L664-L672) (verified)
  - *To reach the next level:* Default is not a read-only tool set.
- **B L1:** A misused tool reaches the whole tenant: bulk deletes by ID list, bulk quarantine action by filter, RTR commands on any host the session can reach; only query limits are bounded. — [falcon_mcp/modules/quarantine.py:296-304](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/quarantine.py#L296-L304); [falcon_mcp/modules/host_groups.py:335-338](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/host_groups.py#L335-L338) (verified)
  - *To reach the next level:* No bounds on destructive batch sizes or per-call target counts.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

The server never executes model-supplied text as code on its own host: no subprocess, shell, eval, exec or deserialisation calls exist in the package. Model-influenced strings (FQL filters, CQL queries, RTR command strings, workflow parameters) are forwarded to the Falcon API, which runs them in CrowdStrike's cloud or on endpoints under the API client's scopes; those controls are scored under C3 and C5. This is absence of a local execution surface, not a sandbox.

- **Structural absence:** searched `rg -n -S 'subprocess|os\.system|\beval\(|\bexec\(|pickle|__import__|yaml\.load'` in `falcon_mcp` → 0 hits (no local code-execution, shell or unsafe-deserialisation calls anywhere in the package); [falcon_mcp/modules/rtr.py:534-543](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/rtr.py#L534-L543)
- **Notes:** RTR and CQL pass remote command and query strings to the Falcon API; the server itself runs nothing locally. Scored under C3 (validation) and C5 (consequences).

### C5 Untrusted input blast radius — 0.12 (high)

Tool results are plain JSON records from the Falcon API with no provenance or untrusted-content marking, so analyst-controlled or attacker-influenced text (host names, detection command lines, case notes, dark-web content, AI-session prompts, files read through RTR) enters the model with the same standing as anything else. The server's instructions and tool descriptions contain operational guidance only, and the only outbound destination is the Falcon API, so there is no arbitrary-URL egress channel from the server itself. In the default configuration the same session can read sensitive endpoint data and call destructive tools, with nothing in the server stopping a hijacked model; the opt-in read-only mode removes the state-change leg but nothing marks content as untrusted.

- **S L1:** Outputs are plain records with no provenance or untrusted flag, and unannotated structure beyond pagination envelopes in a few modules. — [falcon_mcp/modules/base.py:329-340](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/base.py#L329-L340); [falcon_mcp/server.py:49-51](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/server.py#L49-L51) (verified)
  - *To reach the next level:* No separation of returned content from metadata with provenance or untrusted flags the host can act on.
- **C L0:** No untrusted source (RTR file contents, case comments, recon content, Guardian prompts, host attributes) is distinguished from other tool output. — [falcon_mcp/modules/rtr.py:501-543](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/rtr.py#L501-L543) (verified)
  - *To reach the next level:* At least some sources would need to be marked or handled differently.
- **D L0:** No injection-containment control exists to be on by default; read-only mode is a separate opt-in. — [falcon_mcp/server.py:73](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/server.py#L73) (verified)
  - *To reach the next level:* Containment would have to exist and be on by default.
- **B L1:** Default session can read private tenant data and call irreversible tools unattended by the server, but the server has no arbitrary egress channel (traffic goes only to the configured Falcon API). — [falcon_mcp/modules/fusion.py:101-110](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/fusion.py#L101-L110); [falcon_mcp/client.py:53-55](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/client.py#L53-L55) (verified)
  - *To reach the next level:* Irreversible tools are not gated or previewed by the server in the default configuration.
- **Cap:** G1 — The only containment-style mode (read-only) is opt-in, so the primary control is off by default.

### C6 Memory, context & configuration integrity — 0.25 (low)

The server keeps no memory or retrieval store and loads no instruction files, but it calls load_dotenv() at startup, so a .env file found by python-dotenv can silently set the Falcon credentials, API base URL and the server's own safety switches (read-only mode, enabled modules, tool lists) read afterwards from the environment. python-dotenv's default search starts from the package location, so exposure depends on where the package is installed or run from. That search behaviour is inferred from the library, not read here. The README also tells users to use a .env file.

- **S L1:** A .env file is auto-loaded silently with no trust prompt and can change endpoint, credentials and safety defaults. — [falcon_mcp/server.py:682](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/server.py#L682); [falcon_mcp/server.py:651](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/server.py#L651) (verified)
  - *To reach the next level:* Security-relevant settings should come only from explicit user scope or flags.
- **C L1:** One configuration-load path exists and it is not controlled. — [falcon_mcp/client.py:53-55](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/client.py#L53-L55) (inferred)
  - *To reach the next level:* Needs a trust decision for workspace-supplied files.
- **D L2:** No shared memory exists, but the config load is not isolated from workspace files by default (dotenv search path inferred). — [falcon_mcp/server.py:682](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/server.py#L682) (inferred)
  - *To reach the next level:* Config should only come from user scope.
- **B L1:** A planted .env persists for every later launch and can redirect credentials to another host or widen the tool surface. — [falcon_mcp/client.py:53-55](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/client.py#L53-L55) (inferred)
  - *To reach the next level:* Would need a trust prompt or user-scope-only loading.
- **Cap:** C6-REPOCONFIG — load_dotenv() auto-loads a .env that can redirect the Falcon base URL and credentials and switch off read-only mode without a trust decision (search location inferred from python-dotenv defaults).

### C7 Third-party extensions — 1.00 (high)

The server loads no third-party code at runtime: its module registry imports only packages under falcon_mcp.modules, and there is no plugin loader, MCP client, package installer or downloaded model. Build dependencies are out of scope here.

- **Structural absence:** searched `rg -n -S 'entry_points|iter_entry_points|load_plugin|subprocess'` in `falcon_mcp` → 0 hits (no plugin, entry-point or process-launching code); [falcon_mcp/registry.py:44-58](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/registry.py#L44-L58)

### C8 Secrets & sensitive-data protection — 0.35 (high)

Credentials come from environment variables (or constructor arguments) and are held in plain strings with no masking, secret store or redaction layer. The server has no telemetry, and default logging is limited to start-up and policy messages on stderr. With --debug, request parameters and bodies are logged unredacted. The optional HTTP API key can be given on the command line, which exposes it in the process list. RTR file reads can place file contents, including secrets on endpoints, into model context. The API client secret is long-lived but scoped to the operator's grants and rotatable in Falcon.

- **S L1:** Secrets are read from environment variables into plain attributes; there is no masking, secret store or output scanning. — [falcon_mcp/client.py:51-52](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/client.py#L51-L52); [falcon_mcp/server.py:616-620](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/server.py#L616-L620) (verified)
  - *To reach the next level:* No type-level masking or redaction of tool output or logs.
- **C L1:** No protected path beyond not logging credentials in normal operation. — [falcon_mcp/modules/base.py:267](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/base.py#L267); [falcon_mcp/modules/rtr.py:394](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/rtr.py#L394) (verified)
  - *To reach the next level:* Debug logs, model-bound output and error messages are not redacted.
- **D L2:** No telemetry exists, and default log level logs no payloads; redaction does not exist so debug output is verbose by choice. — searched `rg -n -S 'sentry|posthog|opentelemetry|analytics'` in `falcon_mcp/client.py falcon_mcp/server.py falcon_mcp/common falcon_mcp/modules/base.py` → 0 hits (no telemetry SDKs in core files); [falcon_mcp/common/logging.py:22](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/common/logging.py#L22) (verified)
  - *To reach the next level:* Redaction would need to be always on.
- **B L2:** Leaked API client secret is scoped to the operator's grants but long-lived, until rotated in Falcon. — [falcon_mcp/client.py:60](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/client.py#L60); [falcon_mcp/client.py:51-52](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/client.py#L51-L52) (verified)
  - *To reach the next level:* Credentials are not short-lived or per-task.
- **Cap:** none

### C9 Audit & traceability — 0.07 (high)

The server keeps no record of tool calls. Normal logging covers start-up, policy and authentication messages; the only per-call logging of the operation and its parameters is at debug level in a handful of shared helpers, written to stderr with no caller or approver attribution. HTTP access logs from the web server record requests but not tool names or arguments. Falcon's own tenant audit trail will show the API client and the falcon-mcp user agent, but that is the vendor's record, not the server's.

- **S L0:** Tool calls are logged only at debug level and only in some shared helpers. — [falcon_mcp/modules/base.py:267](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/base.py#L267); [falcon_mcp/modules/base.py:309](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/base.py#L309) (verified)
  - *To reach the next level:* No structured per-call record of arguments, status and time at default level.
- **C L1:** Debug logging covers the shared search and query helpers but not every operation, and not tool invocation itself. — [falcon_mcp/modules/base.py:364](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/base.py#L364); [falcon_mcp/modules/rtr.py:394](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/rtr.py#L394) (verified)
  - *To reach the next level:* No audit record for every tool call.
- **D L0:** Per-call logging is debug-only, which is opt-in. — [falcon_mcp/server.py:575](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/server.py#L575) (verified)
  - *To reach the next level:* Needs to be on at default level.
- **B L0:** Logging is best-effort to stderr with no durability or fail-closed behaviour for destructive actions. — [falcon_mcp/common/logging.py:25-29](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/common/logging.py#L25-L29) (verified)
  - *To reach the next level:* Records per action with errors surfaced; destructive tools should not run without a written record.
- **Cap:** none

### C10 Limits & kill switch — 0.38 (high)

The server bounds some of its own work: list and search limits have schema ceilings, RTR waits are capped at 600 seconds, and the polling tools for NGSIEM and AgentWorks have deadlines (300 and 45 seconds by default, operator-configurable by environment variable). Concurrent tool calls share a default pool of 40 worker threads, which is incidental backpressure rather than a stated rate limit. There is no rate limit on write tools, no cap on delete batch size, no kill switch, and a cancelled or timed-out request keeps its worker thread until the blocking Falcon call returns, so in-flight actions are not interrupted.

- **S L2:** Server-enforced ceilings exist on limits and on polling waits for some tools. — [falcon_mcp/modules/rtr.py:596-601](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/rtr.py#L596-L601); [falcon_mcp/modules/ngsiem.py:48](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/ngsiem.py#L48) (verified)
  - *To reach the next level:* No caps or concurrency/rate limits on every operation, and no cancellation of in-flight work.
- **C L1:** Limits are in the tool handlers that poll or search; there is no top-level budget or timeout wrapper around every call. — [falcon_mcp/modules/base.py:50-60](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/base.py#L50-L60) (verified)
  - *To reach the next level:* Timeouts do not wrap every Falcon call.
- **D L2:** Sensible defaults exist, operator-configurable, but the model can choose values up to the schema ceilings. — [falcon_mcp/modules/ngsiem.py:47-48](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/ngsiem.py#L47-L48); [falcon_mcp/modules/agentworks.py:62](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/agentworks.py#L62) (verified)
  - *To reach the next level:* Model cannot raise limits and ceilings cannot be exceeded by config.
- **B L1:** No ceiling on bulk destructive calls, and stopping does not interrupt running blocking calls or asynchronous work already started in Falcon. — [falcon_mcp/modules/base.py:56-60](https://github.com/CrowdStrike/falcon-mcp/blob/9bc0efeb92e38ca0667e5a18847a7cbf09dcd206/falcon_mcp/modules/base.py#L56-L60) (verified)
  - *To reach the next level:* Spend and action ceilings and cancelable in-flight calls.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Falcon records and RTR output returned unmarked, falcon_mcp/modules/rtr.py:501 · [B] sensitive data/systems: EDR host, detection and log data, falcon_mcp/modules/hosts.py:62 · [C] state change / egress: write and delete tools registered by default, falcon_mcp/modules/base.py:126 · Same default session? Yes

## Highest-impact improvements
1. Make read-only the default, requiring an explicit flag to register mutating tools. — C2 D L2→L3, +0.050 before caps (Playbook 5)
2. Write a structured audit record at INFO for every tool call (tool, arguments, status, time, caller). — C9 S L0→L2, +0.150 before caps (Playbook 1 step 3)
3. Stop auto-loading .env from discovered locations; accept only an explicit --env-file or the process environment. — C6 S L1→L3, +0.150 before caps (Playbook 2)
4. Tag returned records with provenance and an untrusted flag so the host can act on them. — C5 S L1→L3, +0.150 before caps (Playbook 1)
5. Validate RTR base_command against an explicit read-only allowlist in code. — C3 S L2→L3, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The behaviour of python-dotenv's file search (C6) and of FalconPy (debug logging, default HTTP timeouts, scope enforcement) is inferred from library behaviour and was not read here.
- Annotation accuracy was checked by scanning operation names and registration blocks across modules, not by reading every one of roughly 140 tool handlers.
- No reviewer-directed text was found in the repository; the only hit for the search terms was an unrelated test comment.
- C4 and C7 are structural absence of local execution and third-party loading; the unweighted applicable score is reported alongside the total.
- Docs, deployment guides and the unit and integration tests were only skimmed.
