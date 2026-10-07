# Defense-in-Depth Score: Google Security MCP servers

**Repo:** https://github.com/google/mcp-security · **Commit:** `9885ec6856ec72333091cf1a3b2ac1bb26abe149` · **Reviewed:** 2026-10-03
**What it is:** MCP servers for Google SecOps (Chronicle), SecOps SOAR, Threat Intelligence and Security Command Center
**Category:** Cybersecurity
**Scored configuration:** Local stdio servers (secops, scc, gti, secops-soar with no --integrations) launched as the README shows, using the operator's Google ADC, service-account key, VirusTotal key or SOAR app key, with no host-side approval layer assumed.
**Agent surface (default):** code execution opt-in · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication opt-in

## Score: 2.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C2 | Approval gates | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C3 | Tool & action scoping | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L0 | 0.07 | C5-WORSTCASE | **0.07** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L0 | L1 | 0.20 | — | **0.20** | Medium |
| C7 | Third-party extensions | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C9 | Audit & traceability | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |


These are thin tool servers over powerful security-operations APIs, and the code itself adds almost no safety layer: no read-only mode, no risk annotations, no confirmation or dry-run for deletes and alert changes, no per-call audit record, and no limits beyond a few page-size caps. Every call runs with whatever Google credential, SOAR key or VirusTotal key the operator supplied, and the model can choose the target project, customer and region. The strongest control is that SOAR integrations (including remote command, email and containment actions) are off unless the operator names them. A hijacked session can delete feeds, rewrite alerts and upload any local file to VirusTotal with no server-side check.

## Critical gaps
- GTI analyse_file opens any local path the process can read and uploads it to VirusTotal, where it is shared with the community, with no path validation or confirmation. (ASI01, ASI02, LLM02; C5) — [server/gti/gti_mcp/tools/files.py:263](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/gti/gti_mcp/tools/files.py#L263)
- In the default configuration one session can read private SIEM data and untrusted text and also run irreversible deletes, with no approval, annotation or dry-run in the servers. (ASI01, ASI09, LLM06; C5) — [server/secops/secops_mcp/tools/feed_management.py:500](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/tools/feed_management.py#L500)

## Criterion details

### C1 Identity & least privilege — 0.05 (high)

Each server acts with one static credential the operator provides: Google Application Default Credentials (or an optional service-account key path) for SecOps and SCC, a VirusTotal key for GTI, and a SOAR app key. Nothing in the code narrows that authority: read and write tools share the credential, there is no read-only identity, and no per-request authorization. The SecOps tools also accept project, customer and region arguments from the caller and build a fresh client for each call, so the model chooses which tenant the credential is applied to. The SOAR key can drive actions across every integration instance the tenant has configured.

- **S L0:** SecOps and SCC fall back to Application Default Credentials (the SCC source comment names gcloud user login) and SOAR/GTI use a single long-lived key from the environment; no code scopes any of them. — [server/secops/secops_mcp/server.py:71-75](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/server.py#L71-L75); [server/scc/scc_mcp.py:45](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/scc/scc_mcp.py#L45); [server/secops-soar/secops_soar_mcp/bindings.py:60-62](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/bindings.py#L60-L62) (verified)
  - *To reach the next level:* No dedicated narrowed identity, no read-only credential split from write tools, and no per-tool scoping (L1 needs a dedicated but broad identity enforced in code).
- **C L0:** Every SecOps tool builds its own client via get_chronicle_client with caller-supplied project_id, customer_id and region, and no tool performs an authorization check. — [server/secops/secops_mcp/server.py:45-63](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/server.py#L45-L63); [server/secops/secops_mcp/tools/feed_management.py:499-500](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/tools/feed_management.py#L499-L500) (verified)
  - *To reach the next level:* A shared layer that authorizes each call against a least-privilege policy before the credential is used.
- **D L0:** There is no narrower default: the server uses whatever role the credential has, and least privilege depends entirely on the operator hardening the credential outside the repo; unset tenant variables fall back to hard-coded default IDs. — [server/secops/secops_mcp/server.py:38-41](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/server.py#L38-L41); searched `rg -n 'read_only_mode|readonly_mode|read-only mode'` in `server/secops/secops_mcp server/scc server/gti/gti_mcp` → 0 hits (no read-only mode flag in SecOps, SCC or GTI code) (verified)
  - *To reach the next level:* A read-only default identity with write access requiring explicit operator elevation.
- **B L1:** If authorization fails (it does not exist), the credential can write to SIEM rules, feeds, watchlists and alerts, mute SCC findings, and run SOAR actions across every enabled integration; no approval layer in this repo survives. — [server/scc/scc_mcp.py:846](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/scc/scc_mcp.py#L846); [server/secops/secops_mcp/tools/security_alerts.py:289](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/tools/security_alerts.py#L289) (verified)
  - *To reach the next level:* Credentials limited to one project with mostly read access and non-destructive writes.
- **Cap:** none
- **Notes:** ADC use is the operator's own authority; scored as ambient because the code never narrows it. Remote hosted variant (docs/remote_server.md) uses IAM roles but is a Google-hosted service outside this code.

### C2 Approval gates — 0.05 (high)

The servers expose reads and writes as mostly separate tools, but none carries a risk annotation (no read-only or destructive hints anywhere in the code), none offers a dry run or confirmation step, and there is no server-enforced read-only mode. Destructive tools such as feed deletion, data-table row deletion, watchlist deletion, alert rewriting and SCC mute execute immediately when called. The SOAR server mixes hundreds of action wrappers (including remote command, email and containment actions when enabled) with no signalling of which ones change state. The host is left to decide what to gate with no help from the server.

- **S L0:** No tool in any server sets read-only or destructive hints, and mutating and read-only SOAR action wrappers are registered identically. — searched `rg -n -S 'readOnlyHint|destructiveHint|ToolAnnotations'` in `server run-with-google-adk/src` → 0 hits (no annotation use in any server or the ADK runner); [server/secops/secops_mcp/tools/feed_management.py:450-451](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/tools/feed_management.py#L450-L451) (verified)
  - *To reach the next level:* Accurate read-only/destructive annotations on every tool (L2 requirement).
- **C L0:** The most consequential tools (delete_feed, delete_data_table_rows, do_update_security_alert, set_finding_mute, and the opt-in SOAR command and HTTP-request actions) have no gate or confirmation of any kind in the server. — [server/secops/secops_mcp/tools/data_table_management.py:377](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/tools/data_table_management.py#L377); [server/secops-soar/secops_soar_mcp/marketplace/ssh.py:596](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/marketplace/ssh.py#L596) (verified)
  - *To reach the next level:* At least a flag that routes consequential tools through a host-enforceable confirmation (L1 needs partial flagging).
- **D L0:** Approval is not a feature of the servers at all; it is left entirely to the MCP host and the ADK runner configures none. — [run-with-google-adk/src/mcp_security_agent/agent.py:54-59](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/run-with-google-adk/src/mcp_security_agent/agent.py#L54-L59); [run-with-google-adk/src/mcp_security_agent/callbacks.py:35-36](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/run-with-google-adk/src/mcp_security_agent/callbacks.py#L35-L36) (verified)
  - *To reach the next level:* A server-side confirmation or read-only switch that is on by default.
- **B L1:** Feed, data-table row and watchlist deletions and public VirusTotal uploads cannot be undone by the server, while alert and mute changes are reversible; there are no previews, undo, or rate limits on consequential actions. — [server/secops/secops_mcp/tools/feed_management.py:459](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/tools/feed_management.py#L459); [server/gti/gti_mcp/tools/files.py:255](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/gti/gti_mcp/tools/files.py#L255) (verified)
  - *To reach the next level:* Dry-run or preview for destructive operations plus checkpoints and rate limits.
- **Cap:** none
- **Notes:** C2-POWERBYPASS not applied: the opt-in shell-like SOAR tools are absent from the default configuration (no --integrations means none are registered). Any host-side approval is not credited.

### C3 Tool & action scoping — 0.38 (high)

Tools use typed Python signatures, and a few validate arguments: SCC mute only accepts two values, and every SOAR action checks its scope argument against the list the SOAR server returns. Most free-form arguments are passed straight through, though: raw filters, UDM and YARA-L text, case IDs interpolated into URL paths, and a local file path that GTI opens and uploads to VirusTotal with no path check. SOAR integrations are selected by an operator allowlist and none load by default, which is the main scoping control. Other tools, including all SecOps write tools and the SOAR case tools, are always on.

- **S L2:** Typed signatures and a few real checks exist (mute enum, SOAR scope membership), but file paths, filters and URL path segments are passed through without containment or encoding. — [server/secops-soar/secops_soar_mcp/marketplace/csv.py:45-50](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/marketplace/csv.py#L45-L50); [server/gti/gti_mcp/tools/files.py:263](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/gti/gti_mcp/tools/files.py#L263); [server/scc/scc_mcp.py:160-168](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/scc/scc_mcp.py#L160-L168); [server/secops-soar/secops_soar_mcp/case_management.py:186](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/case_management.py#L186) (verified)
  - *To reach the next level:* Allowlist validation in code for paths, filters and identifiers (L3).
- **C L1:** Validation is shallow and uneven: the scope check repeats across SOAR wrappers but covers only that one argument, and the SecOps, GTI and case tools do not validate content. — searched `rg -n 'if scope not in bindings.valid_scopes'` in `server/secops-soar/secops_soar_mcp/marketplace` → 2123 hits (one scope-membership check per SOAR wrapper; validates a single argument only); [server/scc/scc_mcp.py:807](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/scc/scc_mcp.py#L807) (verified)
  - *To reach the next level:* Substantive validation on most tools through a shared layer (L2 to L3).
- **D L2:** Tool groups are selectable: SOAR integrations load only when named via --integrations and the ADK runner's per-server flags default to false, but the default group still includes write tools and the SecOps, SCC and GTI servers register everything. — [server/secops-soar/secops_soar_mcp/server.py:100-101](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/server.py#L100-L101); [server/secops-soar/secops_soar_mcp/server.py:35-38](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/server.py#L35-L38); [run-with-google-adk/src/mcp_security_agent/config.py:38](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/run-with-google-adk/src/mcp_security_agent/config.py#L38) (verified)
  - *To reach the next level:* A read-only default tool set with write tools requiring explicit enabling (L3).
- **B L1:** Misused tools can reach any project or customer the credential covers, upload any readable local file to a public service, and (with integrations enabled) send arbitrary HTTP requests via the SOAR HTTP and Google Cloud API actions. — [server/secops-soar/secops_soar_mcp/marketplace/httpv2.py:102](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/marketplace/httpv2.py#L102); [server/gti/gti_mcp/tools/files.py:252](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/gti/gti_mcp/tools/files.py#L252) (verified)
  - *To reach the next level:* Scoped, quantity-bounded tools without general-purpose request tools.
- **Cap:** none
- **Notes:** Scope check wraps one argument in each SOAR wrapper; 2123 count equals the number of such checks in the marketplace directory. Opt-in integrations not scored against the default group.

### C4 Code-execution isolation — 0.05 (high)

The server processes themselves contain no local command or code execution path, so nothing runs on the MCP host. The opt-in SOAR wrappers do forward model-supplied commands and scripts to remote endpoints through the tenant's SOAR integrations (SSH run-command, runner commands, CrowdStrike scripts), and these have no sandbox, filtering or approval in this repo. In the default configuration none of those wrappers is registered, because integrations must be named by the operator. Isolation is therefore absent rather than weak, and its blast radius is whichever managed endpoints the operator's SOAR instances reach.

- **S L0:** No isolation primitive exists; the remote command and script wrappers pass text straight to SOAR actions that run it on managed hosts. — [server/secops-soar/secops_soar_mcp/marketplace/ssh.py:596](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/marketplace/ssh.py#L596); [server/secops-soar/secops_soar_mcp/marketplace/crowdstrikefalcon.py:266](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/marketplace/crowdstrikefalcon.py#L266); [server/secops-soar/secops_soar_mcp/marketplace/runners.py:28](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/marketplace/runners.py#L28) (verified)
  - *To reach the next level:* A boundary the model cannot redefine around remote command execution (L2 or higher).
- **C L0:** None of the remote execution wrappers is sandboxed; the only local-exec search over server code is empty, so the exposure is exactly these opt-in wrappers. — searched `rg -n 'subprocess|os\.system|eval\(|exec\(|Popen' --glob '*.py' --glob '!tests/**' --glob '!marketplace/**'` in `server run-with-google-adk/src` → 4 hits (all 4 hits are log messages or a prompt sentence mentioning the word subprocess/telemetry in the ADK runner; no local execution code) (verified)
  - *To reach the next level:* Every execution path behind a boundary (L3).
- **D L0:** No sandbox exists to be on by default; the execution wrappers are present only when an operator names the matching integration. — [server/secops-soar/secops_soar_mcp/server.py:100-101](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/server.py#L100-L101) (verified)
  - *To reach the next level:* A default-on boundary (L2 or higher).
- **B L1:** Commands reach managed endpoints chosen by the operator's SOAR integration instances, not the MCP host, but nothing limits what runs there or with which privileges. — [server/secops-soar/secops_soar_mcp/marketplace/ssh.py:596](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/marketplace/ssh.py#L596) (verified)
  - *To reach the next level:* Workspace-only reach with no secrets and ephemeral execution (L3 to L4).
- **Cap:** none
- **Notes:** Default configuration has no execution wrapper registered; scored 0 rather than structurally absent because the repository ships the wrappers and the README example enables integrations by flag.

### C5 Untrusted input blast radius — 0.07 (high)

These servers read content an attacker can influence: log events, alert and case text, comments, threat-intelligence records and finding descriptions. Outputs come back as plain JSON or text with no provenance flag or untrusted marker, and nothing in the server limits what a hijacked session can then do. Several tool descriptions also carry long next-step guidance telling the model what to do after reading results. A manipulated session can delete feeds, rewrite alerts, ingest forged logs or upload a local file to a public service, all unattended at the server level.

- **S L1:** Results are returned as plain dictionaries or JSON strings with no provenance or untrusted flag, and tool descriptions include next-step workflow guidance. — [server/secops/secops_mcp/tools/security_alerts.py:211-212](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/tools/security_alerts.py#L211-L212); [server/secops/secops_mcp/tools/security_alerts.py:293](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/tools/security_alerts.py#L293) (verified)
  - *To reach the next level:* Structured outputs separating content from metadata with a provenance flag (L2 to L3).
- **C L0:** No source of untrusted content (events, comments, case text, TI records) is distinguished from any other output. — [server/secops-soar/secops_soar_mcp/case_management.py:186-187](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/case_management.py#L186-L187); searched `rg -n -i 'untrusted|provenance|taint' --glob '*.py' --glob '!tests/**' --glob '!marketplace/**'` in `server run-with-google-adk/src` → 0 hits (no handling of untrusted content or provenance anywhere in server or runner code) (verified)
  - *To reach the next level:* Distinct handling of at least one untrusted source (L1).
- **D L0:** There is no untrusted-input control to be on by default. — [server/gti/gti_mcp/utils.py:119-120](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/gti/gti_mcp/utils.py#L119-L120) (verified)
  - *To reach the next level:* A default-on mode that limits state change or egress after untrusted content is read.
- **B L0:** In the default configuration the same session can read private SIEM data and untrusted text, upload any readable local file to VirusTotal, and take irreversible actions such as feed or data-table row deletion, with no human step in this code. — [server/gti/gti_mcp/tools/files.py:252-264](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/gti/gti_mcp/tools/files.py#L252-L264); [server/secops/secops_mcp/tools/feed_management.py:500](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/tools/feed_management.py#L500) (verified)
  - *To reach the next level:* Egress and irreversible actions forced through approval or disabled once untrusted content is read.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.
- **Notes:** Host-side gates (if any) are not credited. Hosted remote server docs mention optional Model Armor, which is a Google Cloud service outside this code.

### C6 Memory, context & configuration integrity — 0.20 (medium)

The servers keep no memory of their own, but they can write model-influenced text into shared systems that later reads feed back into context: SOAR case comments and descriptions, SIEM alert comments, data tables and reference lists. Those stored values are not validated, marked, or isolated per user, so injected text in one session can resurface for other analysts and sessions. Configuration comes from environment variables and the SOAR server loads an environment file with a default dotenv call; the ADK runner reads a .env from its working directory for endpoint and key settings.

- **S L1:** Model-written case comments and alert comments are stored as-is and read back by get_case_full_details with no validation or provenance. — [server/secops-soar/secops_soar_mcp/case_management.py:185-188](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/case_management.py#L185-L188); [server/secops-soar/secops_soar_mcp/case_management.py:905-906](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/case_management.py#L905-L906) (verified)
  - *To reach the next level:* Gated or validated writes with provenance marking on stored entries (L2 to L3).
- **C L1:** No stored-content path (cases, alerts, data tables, reference lists) is controlled; the only auto-loaded file is configuration read through dotenv and pydantic settings. — [server/secops-soar/secops_soar_mcp/bindings.py:25](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/bindings.py#L25); [run-with-google-adk/src/mcp_security_agent/config.py:24](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/run-with-google-adk/src/mcp_security_agent/config.py#L24) (verified)
  - *To reach the next level:* Controls on the main stores of model-written content (L2).
- **D L0:** Cases and SIEM objects are tenant-wide shared stores with no per-user or per-session namespace in the code. — [server/secops-soar/secops_soar_mcp/case_management.py:28-29](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/case_management.py#L28-L29) (verified)
  - *To reach the next level:* Per-user or per-session namespaces enforced in queries (L2).
- **B L1:** Stored comments and descriptions persist across sessions and users and re-enter the model through read tools, which can then trigger tool use (inferred from the read and write tools both existing). — [server/secops-soar/secops_soar_mcp/case_management.py:905-906](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/case_management.py#L905-L906) (inferred)
  - *To reach the next level:* Session-scoped or reviewed memory with rollback (L3 to L4).
- **Cap:** none
- **Notes:** C6-REPOCONFIG not applied: dotenv.load_dotenv() with no path searches upward from the calling module's directory in non-interactive use (library behaviour inferred), not the caller's working directory, and the ADK runner reads .env from its own project directory by design. Not verified against dotenv source in this audit.

### C7 Third-party extensions — 0.30 (high)

The servers do not load outside plugins or models. SOAR marketplace modules are in-repo files imported only when the operator names them, and the ADK runner launches the in-repo servers with uv. Nothing pins or hashes dependencies: packages use lower-bound version ranges, no lockfile is committed, and the README launches with uvx from the registry at whatever version is latest. The marketplace modules run in the SOAR server's own process with its key, and launched servers run as the same user.

- **S L1:** Dependencies are lower-bound ranges, no lockfile is committed, and the README's uvx commands resolve the latest published release at each launch. — [server/secops/pyproject.toml:16-19](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/pyproject.toml#L16-L19); [README.md:69](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/README.md#L69); searched `rg --files --glob '*lock*'` in `.` → 0 hits (no lockfile in the repository) (verified)
  - *To reach the next level:* Pinned versions (L2) and integrity checks (L3).
- **C L1:** Only the module-name allowlist for SOAR integrations constrains what loads; MCP servers launched by the runner and packaged dependencies have no verification. — [server/secops-soar/secops_soar_mcp/server.py:100-108](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/server.py#L100-L108) (verified)
  - *To reach the next level:* Verification across all extension types (L2 to L3).
- **D L2:** No integration is enabled unless the operator names it on the command line, and the runner's server flags default to false, but nothing shows the exact code that will run. — [server/secops-soar/secops_soar_mcp/server.py:65-66](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/server.py#L65-L66); [run-with-google-adk/src/mcp_security_agent/toolsets.py:56-60](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/run-with-google-adk/src/mcp_security_agent/toolsets.py#L56-L60) (verified)
  - *To reach the next level:* Consent that shows exact package, command and permissions (L3).
- **B L1:** Launched servers are separate processes under the same user, and marketplace modules run in-process with the SOAR key. — [run-with-google-adk/src/mcp_security_agent/toolsets.py:57-61](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/run-with-google-adk/src/mcp_security_agent/toolsets.py#L57-L61) (verified)
  - *To reach the next level:* Scrubbed environment and per-extension sandboxing (L2 to L3).
- **Cap:** none
- **Notes:** The extension code is first-party; the score reflects missing verification of what resolves at install and launch time, not an observed malicious extension.

### C8 Secrets & sensitive-data protection — 0.30 (high)

Credentials come from environment variables or a service-account file path; the code does not log them, and there is no telemetry or crash reporting. There is no masking or redaction layer, though, and keys are long-lived: a SOAR app key, a VirusTotal API key and Google credentials. The feed secret tool deliberately returns a newly generated secret into the model's context. Error paths log exception text at debug level without scrubbing.

- **S L1:** Secrets are read from environment variables with no masking or secret-manager integration; one tool returns a generated feed secret to the model. — [server/gti/gti_mcp/server.py:35-38](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/gti/gti_mcp/server.py#L35-L38); [server/secops/secops_mcp/tools/feed_management.py:568-569](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/tools/feed_management.py#L568-L569) (verified)
  - *To reach the next level:* Type-level masking and redaction on main paths (L2).
- **C L1:** No protected path was found beyond not logging the key itself; model-bound output, error messages and logs are unredacted. — [server/secops-soar/secops_soar_mcp/http_client.py:72-73](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/http_client.py#L72-L73); searched `rg -n -i 'redact|SecretStr|mask_secret' --glob '*.py' --glob '!tests/**' --glob '!marketplace/**'` in `server run-with-google-adk/src` → 0 hits (no redaction helpers exist) (verified)
  - *To reach the next level:* Protection for logs and transcripts at least (L2).
- **D L2:** No telemetry or crash reporter exists and logging defaults are modest, but there is no redaction to leave on. — searched `rg -n -i 'sentry|posthog|opentelemetry' --glob '*.py' --glob '!tests/**' --glob '!marketplace/**'` in `server run-with-google-adk/src` → 0 hits (no telemetry SDK present) (verified)
  - *To reach the next level:* Redaction always on and stored data minimised (L3 to L4).
- **B L1:** Leaked keys are long-lived and moderately to broadly scoped (SOAR tenant key, VirusTotal key, Google credentials). — [server/secops-soar/secops_soar_mcp/http_client.py:39-43](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops-soar/secops_soar_mcp/http_client.py#L39-L43) (verified)
  - *To reach the next level:* Short-lived, scoped, rotatable credentials (L3).
- **Cap:** none
- **Notes:** Secret material is not routinely placed in prompts; C8-MODELSECRETS not applied.

### C9 Audit & traceability — 0.25 (high)

The SecOps server logs some actions at info level to stderr (a line per call for many tools), the SCC server logs searches with their filters, and the SOAR case tools and the roughly 2,000 marketplace wrappers write no log at all (the wrappers use print for errors). The GTI server sets its logger to error level, so its info-level lines never show. No log records arguments, results, caller identity or approvals, and none ships to a store the model cannot touch. Platform-side audit trails in Google Cloud or SOAR exist but are not part of this code.

- **S L1:** Unstructured info-level log lines exist for some SecOps and SCC calls; SOAR tools and GTI produce no usable record. — [server/secops/secops_mcp/tools/feed_management.py:496](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/tools/feed_management.py#L496); [server/gti/gti_mcp/server.py:25](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/gti/gti_mcp/server.py#L25) (verified)
  - *To reach the next level:* A structured record of every tool call with arguments, status and timestamps (L2).
- **C L1:** Only part of the SecOps and SCC tool surface logs; case tools and all marketplace wrappers do not. — searched `rg -n 'logger\.|logging\.' --glob '*.py'` in `server/secops-soar/secops_soar_mcp/case_management.py server/secops-soar/secops_soar_mcp/marketplace` → 0 hits (no logging in SOAR case tools or wrappers) (verified)
  - *To reach the next level:* Logging on all built-in tools (L2).
- **D L1:** Logging is on by default but goes to the process stderr, which the host or the agent's environment controls, with no separate protected store. — [server/secops/secops_mcp/server.py:31](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/server.py#L31) (verified)
  - *To reach the next level:* Output outside the agent's reach (L2) written by a component the model cannot control (L3).
- **B L1:** Log lines are best-effort stderr output; actions proceed whether or not a record is written. — [server/secops/secops_mcp/tools/feed_management.py:497-500](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/tools/feed_management.py#L497-L500) (verified)
  - *To reach the next level:* Durable per-action records and fail-closed handling of high-risk actions (L3 to L4).
- **Cap:** none
- **Notes:** Platform audit logs (Cloud Audit Logs, SOAR case wall) were not examined and are not credited.

### C10 Limits & kill switch — 0.33 (high)

A few operations have server-side caps, such as a 1000-item page-size ceiling for rule listing and for SCC finding pages, but result counts and time ranges are mostly caller-chosen: SecOps search takes the event count as a parameter, GTI takes a limit argument, and SCC loops until the caller's maximum. There are no rate limits, no concurrency limits, and no cancellation handling in the servers, and VirusTotal file analysis waits for completion with no explicit bound. The ADK runner sets a 60-second stdio timeout on its client side only.

- **S L2:** Server-enforced caps exist on some list operations (page size is clamped to 1000), while other results are bounded only by caller arguments. — [server/secops/secops_mcp/tools/security_rules.py:71-73](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/tools/security_rules.py#L71-L73); [server/scc/scc_mcp.py:183](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/scc/scc_mcp.py#L183) (verified)
  - *To reach the next level:* Caps on every operation plus rate and concurrency limits (L3).
- **C L1:** The limits that exist apply per call; nothing bounds total work across calls or tasks, and no search finds a rate limiter or timeout in server code. — searched `rg -n 'asyncio.wait_for|Semaphore|rate_limit|RateLimit|ClientTimeout' --glob '*.py' --glob '!tests/**'` in `server run-with-google-adk/src` → 0 hits (no timeout, semaphore or rate limiter in server code) (verified)
  - *To reach the next level:* Per-tool timeouts at minimum (L2).
- **D L1:** Defaults exist (100 events, 10 GTI results, 50 SCC findings) but the model can raise them through ordinary arguments. — [server/secops/secops_mcp/tools/security_events.py:36](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/secops/secops_mcp/tools/security_events.py#L36) (verified)
  - *To reach the next level:* Limits the caller cannot raise (L3) with hard ceilings (L4).
- **B L1:** Without rate limits a runaway session can issue unbounded reads and writes, and in-flight GTI analysis waits unbounded. — [server/gti/gti_mcp/tools/files.py:267](https://github.com/google/mcp-security/blob/9885ec6856ec72333091cf1a3b2ac1bb26abe149/server/gti/gti_mcp/tools/files.py#L267) (verified)
  - *To reach the next level:* Tight per-run ceilings and cancellation of pending calls (L3).
- **Cap:** none
- **Notes:** aiohttp and vt-py library default timeouts were not examined.

## Rule-of-Two check
[A] untrusted input: SIEM events, alert and case text, TI records read via server/secops/secops_mcp/tools/security_events.py:36 · [B] sensitive data/systems: Chronicle, SCC and SOAR tenant data under one static credential (server/secops/secops_mcp/server.py:71-75) · [C] state change / egress: Deletes, alert updates, public upload at server/gti/gti_mcp/tools/files.py:263 · Same default session? Yes

## Highest-impact improvements
1. Add accurate read-only and destructive annotations to every tool, starting with the SecOps and SCC write tools. — C2 S L0→L2, +0.150 before caps (Playbook 5)
2. Offer a server-enforced read-only mode (flag that does not register write tools) and make it the default. — C3 D L2→L3, +0.050 before caps (Playbook 3 step 1)
3. Validate the GTI analyse_file path against an operator-configured directory and block symlink escapes. — C3 S L2→L3, +0.075 before caps (Playbook 3)
4. Write a structured per-call audit record (tool, arguments, status, time, identity) for every server including SOAR wrappers. — C9 S L1→L2, +0.075 before caps (Playbook 1 step 3)
5. Stop accepting project, customer and region from the model; bind them to operator configuration. — C1 C L0→L2, +0.150 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, built or probed, and no wrapped CLI or API was called.
- The 295 SOAR marketplace wrapper files (about 2,100 tools) were reviewed by pattern search and by reading representative files (csv, ssh, case_management), not line by line.
- The python-dotenv search-path behaviour behind the C6 reasoning is recalled library behaviour, not verified in this audit; C6 B is marked inferred.
- Platform-side controls (Google Cloud IAM and audit logs, SOAR permissions, hosted remote server Model Armor) are outside this code and were not credited.
- The ADK runner in run-with-google-adk was reviewed for its tool wiring and config only; its web routes were only skimmed.
- No text aimed at AI reviewers was found in the repository.
