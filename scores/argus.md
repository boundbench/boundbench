# Defense-in-Depth Score: Argus

**Repo:** https://github.com/Sec-Link/Argus-Agentic-SOC-Platform · **Commit:** `dd803f061b24fa010ba0f2901b1902c42537ba8d` (v0.8-1-gdd803f0) · **Reviewed:** 2026-10-03
**What it is:** Open-source AI-native agentic SOC platform (alerts, ticketing, CMDB, workflow automation, AI assistant)
**Category:** Cybersecurity
**Scored configuration:** Docker Compose / Kubernetes deployment as shipped (backend Django API, default settings, auto-approved read-only self-registration), scoring the AI assistant chat and ticket-assistant paths together with the SOAR workflow action engine.
**Agent surface (default):** code execution no · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents no · external communication yes

## Score: 4.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L2 | L1 | 0.45 | C1-PASSTHRU | **0.25** | High |
| C2 | Approval gates | L1 | L0 | L0 | L1 | 0.12 | C2-POWERBYPASS | **0.12** | High |
| C3 | Tool & action scoping | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C6 | Memory, context & configuration integrity | L1 | L0 | L0 | L1 | 0.12 | — | **0.12** | High |
| C7 | Third-party extensions | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L2 | 0.45 | — | **0.45** | High |

Controls where a risk surface exists: 3.18 / 9.0 (35%); 1 criterion scored SA (surface absent).

The AI assistant itself is tightly bounded: its default tool set is four read-only knowledge/skill tools plus four read-only built-in ticket and asset lookups, so a manipulated model cannot change state or reach the network by design. The larger risks sit around it. The SOAR workflow engine runs containment, email and generic HTTP actions with no per-run approval step, shared skill instructions that steer the assistant can be edited by any non-readonly user, and one assistant endpoint forwards the caller's platform token to caller-chosen MCP endpoints. Access control is flat (any authenticated user), and self-registration is auto-approved with read access by default.

## Critical gaps
- The ticket mention endpoint attaches the caller's platform token to requests sent to MCP endpoints named by the caller. (ASI03, T9; C1) — [backend/tickets/views.py:859-862](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/tickets/views.py#L859-L862); [backend/ai_assistant/mcp_gateway.py:231-240](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/mcp_gateway.py#L231-L240)
- The generic HTTP and containment workflow actions run without any approval gate in the default configuration. (ASI02, ASI09; C2) — [backend/workflows/prefect/actions/registry.py:27-33](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/actions/registry.py#L27-L33); [backend/workflows/prefect/flow.py:197](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/flow.py#L197)

## Criterion details

### C1 Identity & least privilege — 0.25 (high)

Everything runs under the platform's own user model: per-user API tokens, a built-in read-only role that the middleware uses to block writes, and a separately restricted credential for the workflow worker that can only reach three named routes. Authorization is otherwise flat: the default permission class is 'any authenticated user', so any non-readonly account can edit and run workflows, change skills and register MCP servers, and any account can read all tickets. By default self-registration is auto-approved with read access. One ticket-assistant endpoint forwards the caller's platform token to MCP endpoints the caller names, which is a token-passthrough pattern.

- **S L2:** Role-scoped identities exist (read-only role, route-limited worker credential) but the default permission class only checks that the user is authenticated. — [backend/siem_project/settings.py:239-241](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/siem_project/settings.py#L239-L241); [backend/workflows/worker_auth.py:18-22](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/worker_auth.py#L18-L22); [backend/accounts/services.py:684-693](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/accounts/services.py#L684-L693) (verified)
  - *To reach the next level:* No per-capability credentials or per-request scoped tokens; read and write share one user token.
- **C L2:** The assistant's built-in tools run on endpoints that require the caller's token, but the check is authentication only, and ticket and asset reads are not scoped to the requesting user. — [backend/ai_assistant/mcp_protocol_views.py:95-99](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/mcp_protocol_views.py#L95-L99); [backend/ai_assistant/views.py:224-226](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/views.py#L224-L226) (verified)
  - *To reach the next level:* No per-principal authorization on tickets or assets, and the ticket assistant passes the caller's token onward to caller-named endpoints.
- **D L2:** Self-registration is auto-approved and yields a read-only account, so the default role is write-denied but readable across all SOC data; widening is an operator action in the admin API. — [backend/accounts/models.py:161](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/accounts/models.py#L161); [backend/accounts/services.py:443](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/accounts/services.py#L443) (verified)
  - *To reach the next level:* Auto-approval is on by default and read access to all tickets is granted to any registrant; a near-minimal default would require explicit admin approval.
- **B L1:** If the authorization layer fails, a non-readonly token can write across workflows, tickets, skills and MCP server records and start containment or notification actions against external systems. — [backend/workflows/views.py:174-176](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/views.py#L174-L176); [backend/workflows/views.py:207](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/views.py#L207) (verified)
  - *To reach the next level:* Writes span several systems; L2 would confine a failure to one system or to read access. Surviving independent layers are the encrypted, bound workflow secrets and the admin-controlled HTTP allowlist.
- **Cap:** C1-PASSTHRU — The ticket mention endpoint attaches the caller's own Authorization header to MCP requests sent to caller-supplied endpoints.
- **Notes:** The cap rests on the mention endpoint, where the endpoint list comes from the request body and the token defaults to the caller's header. The main chat endpoint forces the internal MCP URL (built from the request host) and also forwards the caller's header there.

### C2 Approval gates — 0.12 (high)

The assistant has no consequential tools, so it needs no approval step. The platform's consequential actions live in the workflow engine, and there is no approval node, confirmation step or per-run human gate anywhere in it. A human publishes a workflow once, after which runs triggered by tickets, webhooks, schedules or the API execute every step, including containment, email and generic HTTP calls, unattended. The generic HTTP action is also ungated.

- **S L1:** The only human gate is publishing a workflow manifest once; that single decision authorises every later run of its steps. — [backend/workflows/engine.py:27-30](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/engine.py#L27-L30); searched `rg -n -i 'approv' -g '!**/migrations/**' -g '!**/tests/**'` in `backend/workflows backend/workflow_interfaces` → 1 hits (the single hit is a migration-import note in workflows/README.md, not a run-time gate) (verified)
  - *To reach the next level:* No per-call approval showing the exact action and target before a containment, email or HTTP step runs.
- **C L0:** Every action type, including generic HTTP and containment actions, runs without a gate once a workflow runs. — [backend/workflows/prefect/actions/registry.py:27-33](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/actions/registry.py#L27-L33); [backend/workflows/prefect/flow.py:197](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/flow.py#L197) (verified)
  - *To reach the next level:* No tool, including the generic HTTP call, passes through an approval gate.
- **D L0:** No per-run approval exists to be on or off; ticket-event bindings start published workflows automatically. — [backend/workflows/ticket_invocation.py:36-70](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/ticket_invocation.py#L36-L70) (verified)
  - *To reach the next level:* Approval would need to exist and be on by default.
- **B L1:** Unattended runs can email, post to webhooks, call HTTP endpoints with any listed method including DELETE, and block IPs or disable accounts; containment has release and enable counterparts but nothing forces them, and there are no per-run action quotas. — [backend/workflows/prefect/actions/api_call.py:17](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/actions/api_call.py#L17); [backend/workflows/prefect/actions/disable_user.py:35-41](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/actions/disable_user.py#L35-L41) (verified)
  - *To reach the next level:* Irreversible external actions have no rate limits, recipient limits or previews.
- **Cap:** C2-POWERBYPASS — The generic HTTP action and the containment actions run with no approval gate in the default configuration.
- **Notes:** The model cannot start a workflow: none of its registered tools reaches the workflow engine. The score reflects the platform's consequential-action surface, not the assistant.

### C3 Tool & action scoping — 0.38 (high)

The assistant's tools are narrow and read-only, with typed schemas and clamped result limits, and database access goes through the ORM. Validation is uneven elsewhere. The generic HTTP workflow action has a strong target policy (fixed host, admin allowlist, resolved-address checks, no redirects, capped response), but the containment, lookup and webhook actions accept any URL, the skill-reading tool's argument handling is not strict, and a registry-proxy endpoint fetches caller-supplied URLs. All workflow actions are registered by default, including write and network ones.

- **S L2:** Typed tool schemas and bounded limits exist, and the HTTP action is carefully validated, but several actions are unvalidated, and the skill-reading tool's argument handling is not strict. — [backend/workflows/prefect/actions/api_call.py:127-195](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/actions/api_call.py#L127-L195); [backend/workflows/prefect/actions/api_call.py:251](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/actions/api_call.py#L251) (verified)
  - *To reach the next level:* Not every tool does allowlist validation with resolved-path or host containment; DNS is not pinned in the HTTP action.
- **C L1:** Only the HTTP action, the firewall alias action and the built-in lookup tools validate meaningfully; block-IP (generic), disable/enable user, release IP, lookups and webhooks do not confine their targets. — [backend/workflows/prefect/actions/disable_user.py:33](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/actions/disable_user.py#L33); [backend/workflows/prefect/actions/send_notification.py:116-124](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/actions/send_notification.py#L116-L124); [backend/ai_assistant/views.py:144](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/views.py#L144) (verified)
  - *To reach the next level:* Most tools do not validate; a shared policy layer does not exist.
- **D L2:** The assistant's default tools are read-only, but the registry of workflow actions enables write and network actions together with no per-task tool allowlist. — [backend/workflows/prefect/actions/registry.py:26-40](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/actions/registry.py#L26-L40); [backend/ai_assistant/chat_agent.py:164](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/chat_agent.py#L164) (verified)
  - *To reach the next level:* A read-only default across the whole platform would need write and network actions to be enabled explicitly.
- **B L1:** A misused action can post credentialed requests to any URL for several action types, change firewall or account state on configured devices, and send email; only the HTTP action is limited by an admin allowlist. — [backend/workflows/prefect/actions/block_ip.py:63-73](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/actions/block_ip.py#L63-L73) (verified)
  - *To reach the next level:* Broad reach with only partial limits; L2 would scope actions to a project or workspace.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

No model-reachable path interprets text as code. The workflow engine substitutes variables with a regular expression, conditions use a custom evaluator, database access uses the ORM, and the model has no shell, script or package-install tool. The only process-spawning and exec hits are a manual sample script and a docstring.

- **Structural absence:** searched `rg -n -S 'subprocess|os\.system|\beval\(|\bexec\(|pickle|shell=True|__import__|importlib' -g '!**/migrations/**' -g '!**/tests/**'` in `backend` → 3 hits (all three hits are in backend/workflows/sample_workflows (a manual deploy helper and a docstring), not reachable from the API, the assistant or the workflow runtime)

### C5 Untrusted input blast radius — 0.50 (high)

Untrusted content reaches the model in several places: ticket alert JSON, tool results, knowledge-base documents and skill text. Nothing distinguishes it from instructions, but the default assistant paths have only read-only tools, so a manipulated model can read tickets and assets and shape its answer, not change state or call out. One ticket endpoint lets the caller list MCP endpoints whose tools the model may select without approval. Separately, alert fields flow into workflow variables that choose containment targets, which is not model-mediated.

- **S L2:** The default chat and ticket-assistant paths register only read-only tools and force the internal MCP endpoint, which structurally limits a hijacked model, but there is no taint tracking and one endpoint does not force the internal endpoint. — [backend/ai_assistant/chat_agent.py:164-170](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/chat_agent.py#L164-L170); [backend/ai_assistant/views.py:222](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/views.py#L222); [backend/tickets/views.py:843](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/tickets/views.py#L843) (verified)
  - *To reach the next level:* The restriction is a property of the default tool list, not an enforced rule once untrusted content is read; the mention endpoint can select caller-listed MCP endpoints.
- **C L2:** The read-only restriction applies to every content source in the default paths, but sources are not labelled and the mention endpoint is outside the restriction. — [backend/ai_assistant/views.py:258](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/views.py#L258); [backend/ai_assistant/chat_agent.py:500-504](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/chat_agent.py#L500-L504) (verified)
  - *To reach the next level:* No provenance tagging of ticket text, tool results or skill text, and not every assistant endpoint is covered.
- **D L2:** The read-only tool set is code, but the caller can change MCP overrides on the mention endpoint and silently supply endpoints. — [backend/tickets/views.py:828-860](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/tickets/views.py#L828-L860) (verified)
  - *To reach the next level:* The caller can widen the endpoint set without any warning or approval.
- **B L2:** In the default configuration a hijacked assistant can read any ticket or asset and mislead an analyst, but has no write tool, no outbound tool and no way to start a workflow, so only reversible, in-app effects such as saved chat history occur. — [backend/ai_assistant/mcp_protocol_views.py:275-284](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/mcp_protocol_views.py#L275-L284); [backend/ai_assistant/chat_agent.py:484-488](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/chat_agent.py#L484-L488) (verified)
  - *To reach the next level:* Sensitive SOC records are reachable by the model; L3 needs only low-sensitivity data or fully gated egress.
- **Cap:** none
- **Notes:** Workflow runs triggered by alerts substitute alert fields into containment targets; that path is not model-mediated and is covered in C2 and C3.

### C6 Memory, context & configuration integrity — 0.12 (high)

Persistent instruction text steers the assistant. Skill documents on disk are loaded silently into the system prompt for every user, and any non-readonly authenticated user can create or overwrite them through the API, with no validation, approval, versioning or audit record. Skill-name handling is not strict. Saved chat history is per ticket and is only replayed to the model if the client sends it back. Configuration files come from the deployment, not from a workspace the assistant operates on.

- **S L1:** Skill content is written without validation or approval and loaded silently into the system prompt. — [backend/ai_assistant/views.py:402-417](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/views.py#L402-L417); [backend/ai_assistant/chat_agent.py:410](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/chat_agent.py#L410) (verified)
  - *To reach the next level:* Writes are neither logged nor gated, and skills carry no provenance or integrity protection.
- **C L0:** No store is controlled beyond a character filter on the skill name; skills, the knowledge base and saved traces have no write gating. — [backend/ai_assistant/views.py:396-397](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/views.py#L396-L397) (verified)
  - *To reach the next level:* At least the main instruction store would need controlled writes.
- **D L0:** Skills are one global directory shared by every user and every session. — [backend/ai_assistant/skill_library.py:17-18](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/skill_library.py#L17-L18); [backend/ai_assistant/skill_library.py:114-115](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/skill_library.py#L114-L115) (verified)
  - *To reach the next level:* No per-user or per-tenant namespace for skills.
- **B L1:** A poisoned skill persists across all users' sessions and can steer the model's read-only tool calls and its answers, but the model has no write tool to act on it. — [backend/ai_assistant/chat_agent.py:372](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/chat_agent.py#L372) (verified)
  - *To reach the next level:* Persistence is global and silent; it is not easily inspected or purged by reviewers.
- **Cap:** none

### C7 Third-party extensions — 0.30 (high)

The platform loads no third-party code into its own process. MCP is used as a client to remote JSON-RPC endpoints that a user lists or registers; the endpoints, their tool lists and their tokens are not pinned or verified, tool lists are cached for three minutes, and there is no re-approval when definitions change. In the default chat paths the internal endpoint is forced, so registered servers are ignored, but the mention endpoint accepts caller-supplied endpoints and sends the caller's platform token to them.

- **S L1:** Endpoints are user-chosen and unpinned, with no digest, version or definition check. — [backend/ai_assistant/mcp_gateway.py:23](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/mcp_gateway.py#L23); [backend/ai_assistant/models.py:6-10](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/models.py#L6-L10) (verified)
  - *To reach the next level:* No pinning or integrity check of endpoints or their tool definitions.
- **C L1:** Only one extension type exists (remote MCP endpoints) and none of it is verified. — [backend/ai_assistant/mcp_gateway.py:429-460](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/mcp_gateway.py#L429-L460) (verified)
  - *To reach the next level:* Verification applies to no extension type.
- **D L2:** Servers are added by an explicit API call but the person registering sees only a name and URL, any non-readonly user can add one, and nothing is enabled by default. — [backend/ai_assistant/views.py:302-312](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/views.py#L302-L312) (verified)
  - *To reach the next level:* No display of the tool definitions or permissions, and registration is not limited to admin scope.
- **B L1:** A remote MCP endpoint listed on the mention endpoint receives the caller's platform token and ticket text, so a malicious endpoint holds the caller's full API authority; it runs outside the process. — [backend/tickets/views.py:859-862](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/tickets/views.py#L859-L862); [backend/ai_assistant/mcp_gateway.py:231-240](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/mcp_gateway.py#L231-L240) (verified)
  - *To reach the next level:* The endpoint receives the caller's whole token rather than a scrubbed or per-extension credential.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.45 (high)

Workflow credentials are handled well: sensitive fields are write-only, encrypted at rest with a rotatable key ring, bound to their target, decrypted only in the worker, and redacted from results and errors. Startup refuses a placeholder secret key or a missing workflow key outside debug mode. Gaps remain: MCP server tokens are stored in plaintext, request bodies can carry a model API key, and the handling of some stored and deployment credentials is not locked down. No telemetry SDK is present.

- **S L2:** Workflow secrets are Fernet-encrypted with redaction, but MCP tokens are stored in a plaintext column. — [backend/workflows/prefect/secrets.py:18](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/secrets.py#L18); [backend/ai_assistant/models.py:10](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/models.py#L10) (verified)
  - *To reach the next level:* All stored credentials would need encryption at rest and a secret manager.
- **C L2:** Action results and errors are redacted and audit-body logging masks token keys, but MCP token handling is not locked down and workflow outputs are not scanned for secret patterns. — [backend/workflows/prefect/executor.py:28](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/executor.py#L28) (verified)
  - *To reach the next level:* Stored MCP tokens and model-bound content are not covered by the redaction layer.
- **D L2:** No telemetry is shipped and redaction cannot be disabled, but the shipped deployment defaults do not lock down credentials. (verified)
  - *To reach the next level:* Deployment credential defaults are not checked at startup, unlike the secret key.
- **B L1:** Stored firewall, directory and MCP credentials are long-lived and moderately scoped; the model can read none of them, and workflow credentials are decrypted only inside the worker. — [backend/workflows/prefect/executor.py:10](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/executor.py#L10) (verified)
  - *To reach the next level:* Credentials are long-lived and not per-task; rotation requires the operator.
- **Cap:** none

### C9 Audit & traceability — 0.45 (high)

Assistant tool calls are recorded in a database table with tool name, arguments, status, endpoint and timing, and the ticket chat stores the trace. Workflow runs record step inputs, outputs and the user who started them. The record lacks the requesting user on tool calls, the general audit table only logs authentication events and read-only users' activity, and changes to skills, MCP servers and workflows are not audited. Some audit writes are wrapped so failures are swallowed.

- **S L2:** Tool calls are stored with arguments, status and timestamps, but the row has no acting user. — [backend/ai_assistant/models.py:84-90](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/models.py#L84-L90); [backend/ai_assistant/monitoring.py:11-23](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/monitoring.py#L11-L23) (verified)
  - *To reach the next level:* No actor attribution on tool calls and no tamper-evident or exported storage.
- **C L2:** Chat tool calls and gateway calls are recorded, and workflow runs keep step records, but writes to skills, MCP servers and workflow definitions are not audited. — [backend/ai_assistant/chat_agent.py:481](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/chat_agent.py#L481); [backend/accounts/middleware.py:73](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/accounts/middleware.py#L73) (verified)
  - *To reach the next level:* Configuration changes and credential use are not recorded.
- **D L2:** Records are written by application code to the platform database; they sit outside any workspace but the same service can alter or purge them, and a retention job removes old rows. — [backend/siem_project/settings.py:294](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/siem_project/settings.py#L294) (verified)
  - *To reach the next level:* No append-only or off-host storage.
- **B L1:** Audit and monitor writes are best-effort: several are wrapped in broad exception handlers that continue silently. — [backend/ai_assistant/mcp_gateway.py:83](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/mcp_gateway.py#L83); [backend/accounts/middleware.py:88](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/accounts/middleware.py#L88) (verified)
  - *To reach the next level:* Failures are not surfaced and actions proceed without a record.
- **Cap:** none

### C10 Limits & kill switch — 0.45 (high)

The chat loop is capped at six iterations by default and each model call and MCP call has a timeout, but any authenticated caller can raise the iteration count and timeouts per request with no ceiling, and there are no token, cost or total wall-clock limits. The MCP streaming endpoint holds a worker thread open indefinitely. Workflows have a graph-loop guard, a bounded HTTP timeout and response size, and optional per-step timeouts, with cancel forwarded to the workflow engine but errors swallowed; there are no concurrency or rate limits beyond the generic request throttle.

- **S L2:** An iteration cap and per-call timeouts are enforced in code; there is no token, cost or total time cap. — [backend/ai_assistant/chat_agent.py:442](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/chat_agent.py#L442); [backend/ai_assistant/chat_agent.py:147](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/chat_agent.py#L147) (verified)
  - *To reach the next level:* No token or wall-clock cap, and no rate limit on side-effecting actions.
- **C L2:** Limits cover the top-level loop and tool-call timeouts; workflow steps only have optional timeouts and a loop guard. — [backend/workflows/prefect/flow.py:149](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/flow.py#L149); [backend/workflows/prefect/flow.py:192](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/prefect/flow.py#L192) (verified)
  - *To reach the next level:* Spawned workflow runs and background work do not share one budget.
- **D L1:** Defaults exist (six iterations, 45 s) but the requester can raise them per call without a ceiling. — [backend/ai_assistant/chat_agent.py:432-436](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/chat_agent.py#L432-L436); [backend/ai_assistant/chat_agent.py:41](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/chat_agent.py#L41) (verified)
  - *To reach the next level:* No hard ceilings that configuration or callers cannot exceed.
- **B L2:** Moderate ceilings apply per request, but a chat request cannot be cancelled, and cancelling a workflow marks it cancelled even if the engine call fails. — [backend/workflows/views.py:571](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/workflows/views.py#L571); [backend/ai_assistant/mcp_protocol_views.py:308-315](https://github.com/Sec-Link/Argus-Agentic-SOC-Platform/blob/dd803f061b24fa010ba0f2901b1902c42537ba8d/backend/ai_assistant/mcp_protocol_views.py#L308-L315) (verified)
  - *To reach the next level:* No tight cost ceiling, no stop for in-flight chat, and cancel errors are swallowed.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Alert JSON appended to the chat prompt (backend/ai_assistant/views.py:258) · [B] sensitive data/systems: Ticket, CMDB and alert records readable through built-in MCP tools (backend/ai_assistant/mcp_protocol_views.py:95) · [C] state change / egress: None in the default assistant tool set (backend/ai_assistant/chat_agent.py:164); workflow actions are not reachable from the model · Same default session? No

## Highest-impact improvements
1. Add a per-run approval step for containment, email and generic HTTP actions that shows the exact target and payload. — C2 S L1→L3, +0.150 before caps (Playbook 5)
2. Stop forwarding the caller's platform token to caller-supplied MCP endpoints and use the internal endpoint only. — C1 C L2→L3, +0.075 before caps (Playbook 4)
3. Restrict skill writes to an admin role, harden skill-name handling, and record every change. — C6 S L1→L3, +0.150 before caps (Playbook 2)
4. Add a shared URL policy to every workflow action that sends credentials, and harden skill-name handling. — C3 C L1→L3, +0.150 before caps (Playbook 3)
5. Cap iterations, timeouts and total run time server-side and add token limits to the assistant. — C10 D L1→L3, +0.100 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, built or probed.
- Frontend (React) code, the Prefect worker deployment, the Elasticsearch/Kibana integrations, alerts ingestion and detection-rule modules were not reviewed in depth; the review focused on ai_assistant, workflows, accounts, tickets endpoints that call the assistant, settings and deployment manifests.
- The assistant and the SOAR engine are separate paths: the model has no tool that starts a workflow. The workflow engine is scored because it holds the platform's consequential actions; a reader who only cares about the assistant should weigh C2, C3 and C10 accordingly.
- Per-record authorization on tickets and CMDB assets, and the exact permission set of the built-in read-only role, were checked only at the middleware and permission-class level.
- No attempt to steer reviewers was found in README.md, AGENTS.md, docs or the skills directory; AGENTS.md is addressed to coding assistants working on the repo and was treated as data.
