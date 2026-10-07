# Defense-in-Depth Score: Keep

**Repo:** https://github.com/keephq/keep · **Commit:** `465e99152e7586fcbe4c2732213d31c5c28def4b` · **Reviewed:** 2026-10-03
**What it is:** Open-source AIOps and alert management platform with AI correlation and workflow automation that acts on monitoring/ticketing tools
**Category:** Infrastructure & Ops
**Scored configuration:** Shipped docker-compose.yml (AUTH_TYPE=NO_AUTH, file secret manager) with an OpenAI key supplied so the AI incident chat and workflow assistant are enabled.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 2.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L0 | L0 | 0.07 | — | **0.07** | High |
| C2 | Approval gates | L2 | L0 | L1 | L0 | 0.20 | C2-POWERBYPASS | **0.20** | High |
| C3 | Tool & action scoping | L0 | L1 | L0 | L0 | 0.07 | — | **0.07** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C9 | Audit & traceability | L1 | L1 | L2 | L2 | 0.35 | — | **0.35** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |

Controls where a risk surface exists: 1.05 / 9.0 (12%); 1 criterion scored SA (surface absent).

Keep's AI incident chat can call methods on connected providers with no approval step and no allowlist, and the provider-invocation endpoint it uses is not locked down, which widens its reach. Alert text from monitoring tools goes into the same session, so a prompt injection in an alert could act on Kubernetes or ticketing systems and reach stored integration secrets. The default compose file also disables authentication and enables telemetry. Treat the AI features as unsafe to enable on a deployment with real integrations until the invoke endpoint is restricted and chat actions are gated.

## Critical gaps
- The AI chat's invokeProviderMethod runs provider methods with no approval, and the invocation endpoint is not locked down. (ASI02, ASI09; C2) — [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:215-220](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L215-L220)
- A hijacked chat can read secrets and exfiltrate them or take irreversible actions without a human, because untrusted alert text shares the session with ungated provider calls. (ASI01, LLM01; C5) — [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:143-146](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L143-L146); [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:215-220](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L215-L220); [keep/providers/http_provider/http_provider.py:37-39](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/http_provider/http_provider.py#L37-L39)
- Shell and Python provider execution runs unsandboxed in the API container, next to plaintext provider secrets, and is reachable from AI paths. (ASI05; C4) — [keep/providers/bash_provider/bash_provider.py:36-38](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/bash_provider/bash_provider.py#L36-L38); [keep/providers/python_provider/python_provider.py:52](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/python_provider/python_provider.py#L52); [keep/secretmanager/filesecretmanager.py:22-29](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/secretmanager/filesecretmanager.py#L22-L29)
- Default install grants every caller the Admin role, and the agent can use every tenant-wide provider credential. (ASI03; C1) — [docker-compose.yml:21](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/docker-compose.yml#L21); [keep/identitymanager/identity_managers/noauth/noauth_authverifier.py:37-42](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/identitymanager/identity_managers/noauth/noauth_authverifier.py#L37-L42); [keep/providers/gke_provider/gke_provider.py:133-136](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/gke_provider/gke_provider.py#L133-L136)

## Criterion details

### C1 Identity & least privilege — 0.07 (high)

The shipped docker-compose runs with authentication disabled, and in that mode every request is treated as an administrator with every scope. The AI incident chat calls the backend with the user's session, so role checks do apply when authentication is turned on, but provider credentials are stored once per tenant and any installed provider's full credentials can be used through the chat. Gaps in access control on the provider-invocation endpoint also widen what the chat can reach on the API server, which holds every tenant's integration secrets. A hijacked assistant therefore acts with the full authority of every connected system.

- **S L0:** Default AUTH_TYPE is NO_AUTH and the no-auth verifier returns the Admin role (all read/write/delete/execute scopes) for every caller; provider credentials are tenant-wide and unscoped per tool. — [docker-compose.yml:21](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/docker-compose.yml#L21); [keep/identitymanager/identitymanagerfactory.py:55-57](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/identitymanager/identitymanagerfactory.py#L55-L57); [keep/identitymanager/identity_managers/noauth/noauth_authverifier.py:37-42](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/identitymanager/identity_managers/noauth/noauth_authverifier.py#L37-L42); [keep/identitymanager/rbac.py:63](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/identitymanager/rbac.py#L63) (verified)
  - *To reach the next level:* No dedicated, narrower identity for AI-initiated actions; the agent uses the same admin-equivalent session and tenant-wide provider credentials.
- **C L1:** Chat actions go through the backend's scope check (write:providers on the invoke route), but code-executing provider paths then run as the API server process with access to every stored secret. — [keep/api/routes/providers.py:682](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/api/routes/providers.py#L682); [keep/providers/bash_provider/bash_provider.py:36-38](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/bash_provider/bash_provider.py#L36-L38) (verified)
  - *To reach the next level:* Paths that run code on the server bypass any per-principal narrowing because they inherit the server's full authority.
- **D L0:** The shipped compose sets AUTH_TYPE=NO_AUTH, so the default install grants admin to every caller. — [docker-compose.yml:21](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/docker-compose.yml#L21); [keep/identitymanager/identity_managers/noauth/noauth_authverifier.py:37-42](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/identitymanager/identity_managers/noauth/noauth_authverifier.py#L37-L42) (verified)
  - *To reach the next level:* No least-privilege default role; authentication and RBAC require manual hardening.
- **B L0:** Installed providers carry credentials for Kubernetes/GKE (exec in pod, delete pod), ticketing, paging and cloud tools, and the server process can read all of them. — [keep/providers/gke_provider/gke_provider.py:133-136](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/gke_provider/gke_provider.py#L133-L136); [keep/providers/gke_provider/gke_provider.py:376](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/gke_provider/gke_provider.py#L376); [keep/secretmanager/filesecretmanager.py:22-29](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/secretmanager/filesecretmanager.py#L22-L29) (verified)
  - *To reach the next level:* Blast radius would need per-tool scoped or short-lived credentials so a hijack can't reach every integrated system.
- **Cap:** none

### C2 Approval gates — 0.20 (high)

The AI workflow builder asks the user to accept each proposed step before it is added to the canvas, but that only edits an unsaved draft. The AI incident chat, which is the part that acts, has no approval step at all: its invokeProviderMethod, createIncident, enrichment and incident-update actions run as soon as the model calls them. invokeProviderMethod can call action methods on installed providers, such as executing commands in pods, restarting pods, or creating external incidents. Nothing the model does from the chat is shown to a human for approval before it happens.

- **S L2:** The workflow builder uses per-call renderAndWaitForResponse with a step preview the user accepts or rejects. — [keep-ui/features/workflows/ai-assistant/ui/WorkflowBuilderChat.tsx:610](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/features/workflows/ai-assistant/ui/WorkflowBuilderChat.tsx#L610) (verified)
  - *To reach the next level:* Approval UI doesn't show the exact executed call for consequential actions, and there is no argument-level policy or risk tiering.
- **C L0:** The most powerful path, invokeProviderMethod in the incident chat, executes directly from a handler with no confirmation. — [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:194](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L194); [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:215-220](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L215-L220); [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:291](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L291); searched `rg -n 'renderAndWaitForResponse|confirm\('` in `'keep-ui/app/(keep)/incidents/[id]/chat'` → 0 hits (no confirmation step on any incident-chat action) (verified)
  - *To reach the next level:* invokeProviderMethod, createIncident and incident updates need to pass through the same per-call approval as builder steps.
- **D L1:** The builder gate is hard-coded on, but the chat path has no gate to turn on, so consequential actions are effectively unapproved by default. — [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:215-220](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L215-L220); [keep-ui/features/workflows/ai-assistant/ui/WorkflowBuilderChat.tsx:610](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/features/workflows/ai-assistant/ui/WorkflowBuilderChat.tsx#L610) (verified)
  - *To reach the next level:* Approval for consequential chat actions isn't on by default because it doesn't exist.
- **B L0:** Wrongly executed calls include pod exec/delete and external incident creation, with no undo. — [keep/providers/gke_provider/gke_provider.py:133-136](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/gke_provider/gke_provider.py#L133-L136); [keep/providers/gke_provider/gke_provider.py:376](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/gke_provider/gke_provider.py#L376) (verified)
  - *To reach the next level:* Consequential actions have no preview, dry-run or rollback.
- **Cap:** C2-POWERBYPASS — invokeProviderMethod, the most powerful agent action path, executes without any approval in the default configuration.

### C3 Tool & action scoping — 0.07 (high)

The chat's invokeProviderMethod tool takes a provider id, a method name and a free-form parameter object, and the backend passes the parameters straight through. Method and provider resolution on the invoke route is not locked down. The only argument filter found is a substring denylist of metadata hosts in the HTTP provider. In effect the assistant has general-purpose tools rather than narrow, validated ones.

- **S L0:** The invoke route passes the raw request body to provider methods with no typed validation, and method/provider resolution is not locked down. — [keep/providers/bash_provider/bash_provider.py:36-38](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/bash_provider/bash_provider.py#L36-L38) (verified)
  - *To reach the next level:* Method names need an allowlist and arguments need typed validation in code.
- **C L1:** Only the HTTP provider checks its URL, against a substring denylist; the invoke route itself validates nothing. — [keep/providers/http_provider/http_provider.py:20-24](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/http_provider/http_provider.py#L20-L24); [keep/providers/http_provider/http_provider.py:37-39](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/http_provider/http_provider.py#L37-L39) (verified)
  - *To reach the next level:* No shared validation layer applies to invoked provider methods.
- **D L0:** All installed providers are reachable by default, and reachability of further provider types is not locked down. — [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:151-161](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L151-L161) (verified)
  - *To reach the next level:* Write/exec-capable providers need to be off for the assistant unless explicitly enabled.
- **B L0:** A misused tool can run any action method on any connected system, and server-side execution is reachable. — [keep/providers/bash_provider/bash_provider.py:36-38](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/bash_provider/bash_provider.py#L36-L38); [keep/providers/python_provider/python_provider.py:52](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/python_provider/python_provider.py#L52); [keep/providers/gke_provider/gke_provider.py:133-136](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/gke_provider/gke_provider.py#L133-L136) (verified)
  - *To reach the next level:* Tools need to be scoped and quantity-bounded rather than general-purpose.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

Code can run through the bash provider (a shell subprocess) and the Python provider (eval inside the API worker), reachable from workflows and, because access control on the invoke endpoint is not locked down, from AI paths. Neither runs in any sandbox: they execute in the API server container as the same non-root user that holds the database, the file-based secret store and network access. Code that escapes nothing still has everything the server has.

- **S L0:** Shell commands run as a same-user subprocess and Python runs via in-process eval; there is no isolation primitive. — [keep/providers/bash_provider/bash_provider.py:36-38](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/bash_provider/bash_provider.py#L36-L38); [keep/providers/python_provider/python_provider.py:52](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/python_provider/python_provider.py#L52); searched `rg -n -i 'sandbox|seccomp|nsjail|gvisor|firecracker|no-new-privileges|cap_drop'` in `keep docker docker-compose.yml docker-compose.common.yml` → 3 hits (all 3 hits irrelevant: a Graylog test compose env var and a GCP project name default in db_utils.py) (verified)
  - *To reach the next level:* No sandbox (container, OS profile or separate runtime) wraps model-reachable code execution.
- **C L0:** Neither the invoke route nor workflow steps route execution through any sandbox. (verified)
  - *To reach the next level:* Every exec path (invoke route, workflow steps) would need to go through a sandbox.
- **D L0:** No sandbox exists to be enabled; the API image simply runs as a non-root user. — [docker/Dockerfile.api:68](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/docker/Dockerfile.api#L68); searched `rg -n -i 'sandbox|seccomp|nsjail|gvisor|firecracker|no-new-privileges|cap_drop'` in `keep docker docker-compose.yml docker-compose.common.yml` → 3 hits (all 3 hits irrelevant: a Graylog test compose env var and a GCP project name default in db_utils.py) (verified)
  - *To reach the next level:* Sandboxing isn't available, let alone on by default.
- **B L0:** Executed code runs inside the API process environment, which can read the plaintext provider secrets in the state directory and reach the network. — [keep/secretmanager/filesecretmanager.py:22-29](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/secretmanager/filesecretmanager.py#L22-L29); [docker-compose.common.yml:21-22](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/docker-compose.common.yml#L21-L22); [keep/providers/python_provider/python_provider.py:52](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/python_provider/python_provider.py#L52) (verified)
  - *To reach the next level:* Executed code needs to be kept away from secrets and the network.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

The incident chat puts the incident, its alerts and the results of provider calls into the model's context. Alerts arrive from monitoring tools and webhooks, so their names, descriptions and labels are third-party text, and nothing separates that text from the user's instructions. Because the same session can call provider methods without approval, injected text in an alert could make the assistant send data out or take irreversible actions on connected systems. There is no detection or Rule-of-Two control.

- **S L0:** Alerts and tool results go into the copilot context as plain readables; the only guard is the system prompt. — [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:143-146](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L143-L146); [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:139-142](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L139-L142); [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:215-220](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L215-L220) (verified)
  - *To reach the next level:* No code-level restriction applies once untrusted alert content is in context.
- **C L0:** Alert content and provider method results enter context with the same standing as user messages. — [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:143-146](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L143-L146); [keep/api/routes/alerts.py:685-698](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/api/routes/alerts.py#L685-L698) (verified)
  - *To reach the next level:* Untrusted sources aren't distinguished from the principal's input.
- **D L0:** No control exists to be on by default. — [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:143-146](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L143-L146) (verified)
  - *To reach the next level:* Untrusted-content handling isn't present.
- **B L0:** A hijacked chat can exfiltrate data and act irreversibly via provider methods, with no human step. — [keep/providers/http_provider/http_provider.py:37-39](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/http_provider/http_provider.py#L37-L39); [keep/providers/bash_provider/bash_provider.py:36-38](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/bash_provider/bash_provider.py#L36-L38); [keep/providers/gke_provider/gke_provider.py:376](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/gke_provider/gke_provider.py#L376) (verified)
  - *To reach the next level:* Egress and irreversible actions need to require approval once alert content is read.
- **Cap:** C5-WORSTCASE — In the default configuration a hijacked incident chat can both leak secrets and take irreversible actions without a human.

### C6 Memory, context & configuration integrity — 0.00 (high)

Whatever the assistant writes into an incident persists and is fed back to every later chat about that incident, for every user in the tenant. The enrichRCA action appends model text to the incident's root-cause list, and every invokeProviderMethod result is stored as an incident enrichment; the whole incident, including enrichments, is supplied to the model as context. Chat history, including past tool calls and results, is also saved in the browser and reloaded. None of this is validated, labelled as untrusted or gated, so a single injection can persist and steer later sessions into tool use.

- **S L0:** Model output and tool results are written to incident enrichments without validation and re-injected as incidentDetails context. — [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:182-189](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L182-L189); [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:222-226](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L222-L226); [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:139-142](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L139-L142) (verified)
  - *To reach the next level:* Writes to persistent incident context need gating or provenance tagging.
- **C L0:** Neither the enrichment store nor the browser-stored chat history is controlled. — [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:57-63](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L57-L63); [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:67-68](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L67-L68); [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:182-189](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L182-L189) (verified)
  - *To reach the next level:* No persistence path has validation or provenance.
- **D L0:** Incidents and their enrichments are shared across all users of a tenant, and in the default single-tenant NO_AUTH install across every user. — [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:139-142](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L139-L142); [docker-compose.yml:21](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/docker-compose.yml#L21) (verified)
  - *To reach the next level:* Agent-written context isn't namespaced per user or session.
- **B L0:** Poisoned enrichments persist across sessions and users and the chat that reads them can invoke any provider method. — [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:222-226](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L222-L226); [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:215-220](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L215-L220) (verified)
  - *To reach the next level:* Persistent agent-written content needs review before it influences tool use.
- **Cap:** none

### C7 Third-party extensions — 1.00 (high)

Keep does not load third-party code at runtime for its AI features. There is no MCP client, plugin loader, model download or package install path; providers are first-party modules shipped in the image. The search for such loaders found only documentation and developer scripts.

- **Structural absence:** searched `rg -n -i 'modelcontextprotocol|trust_remote_code|pip install|torch\.load|pickle\.load|npx |raw\.githubusercontent'` in `keep keep-ui/app keep-ui/features keep-ui/shared keep-ui/entities keep-ui/widgets` → 10 hits (all 10 hits irrelevant: provider README setup commands, a dev-time protobuf generator script, and a Wazuh-side integration script hint; none is a runtime loader)

### C8 Secrets & sensitive-data protection — 0.15 (high)

Provider credentials are stored by default as plaintext JSON files in the state directory, and API-side secret handling is not locked down. The incident chat only passes provider ids, types and methods to the model, so credentials are not routinely in prompts, but server-side execution paths reachable from the model can read the secret files. Telemetry is on by default: the backend sends usage events to PostHog with a built-in key, the frontend identifies users to PostHog by email, and the compose file sets a Sentry DSN. There is no redaction filter in logging.

- **S L1:** Secrets come from env vars and plaintext files; the chat's provider readable omits credentials, which is the only masking path. — [keep/secretmanager/filesecretmanager.py:22-29](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/secretmanager/filesecretmanager.py#L22-L29); [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:151-161](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L151-L161); searched `rg -n -i 'redact|mask|sensitive|secret'` in `keep/api/logging.py` → 0 hits (no redaction filter in the logging configuration) (verified)
  - *To reach the next level:* No encryption at rest or log redaction.
- **C L1:** Model-bound context excludes credentials, but logs, telemetry and subprocess environments are unprotected. — [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:151-161](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L151-L161) (verified)
  - *To reach the next level:* Logs and telemetry need redaction.
- **D L0:** PostHog is on by default with a hard-coded key, the UI identifies users by email, and Sentry is configured in compose. — [keep/api/core/posthog.py:17-31](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/api/core/posthog.py#L17-L31); [keep-ui/shared/ui/PostHogPageView.tsx:44-48](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/shared/ui/PostHogPageView.tsx#L44-L48); [docker-compose.common.yml:11](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/docker-compose.common.yml#L11) (verified)
  - *To reach the next level:* Telemetry needs to be opt-in by default.
- **B L0:** Long-lived, high-privilege provider credentials sit in files readable by server-side shell and Python execution. — [keep/secretmanager/filesecretmanager.py:22-29](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/secretmanager/filesecretmanager.py#L22-L29); [keep/providers/bash_provider/bash_provider.py:36-38](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/bash_provider/bash_provider.py#L36-L38) (verified)
  - *To reach the next level:* Credentials would need to be scoped, short-lived or kept out of reach of executed code.
- **Cap:** none

### C9 Audit & traceability — 0.35 (high)

When the chat invokes a provider method, the backend logs the provider id and method name, but not the arguments or who asked for it. Incident enrichments go into an audit table that records the user's email, so an AI-made change looks the same as one the user made. The chat conversation itself is kept only in the user's browser. Records go to standard logs and the database, outside any workspace, but code running through the bash provider in the same container could alter the database.

- **S L1:** Invoke logs record provider_id and method only; enrichment audit records the session email without distinguishing AI from human. — [keep/api/routes/providers.py:687-689](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/api/routes/providers.py#L687-L689); [keep/api/routes/incidents.py:1074-1080](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/api/routes/incidents.py#L1074-L1080) (verified)
  - *To reach the next level:* Tool-call records need arguments, result status and agent-vs-human attribution.
- **C L1:** Only the backend invoke and enrich paths leave a server-side trace; chat turns and model decisions are browser-only. — [keep/api/routes/providers.py:687-689](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/api/routes/providers.py#L687-L689); [keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx:57-63](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/(keep)/incidents/[id]/chat/incident-chat.tsx#L57-L63) (verified)
  - *To reach the next level:* Chat turns and every AI action need to be recorded server-side.
- **D L2:** Logging is on by default and lives outside any workspace, but model-reachable code in the API container can alter the database. — [keep/api/routes/providers.py:687-689](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/api/routes/providers.py#L687-L689); [keep/providers/bash_provider/bash_provider.py:36-38](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/bash_provider/bash_provider.py#L36-L38) (verified)
  - *To reach the next level:* Records need to be written by a component the agent's executed code can't touch.
- **B L2:** The invoke log line is emitted before execution, per action, through standard logging. — [keep/api/routes/providers.py:687-689](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/api/routes/providers.py#L687-L689) (verified)
  - *To reach the next level:* No durable, replayable trajectory; arguments are missing.
- **Cap:** none

### C10 Limits & kill switch — 0.20 (high)

Keep's own code sets no step, token or cost limit on the chat assistant, and the API rate limiter is disabled by default. The bash provider has a 60-second default timeout, but the caller sets it, so the model can raise it through the parameters it passes. Stopping the chat in the browser doesn't stop server-side commands already started. CopilotKit's internal loop limits were not examined.

- **S L1:** The only limit is the bash provider's per-call timeout, which is a caller-supplied argument. — [keep/providers/bash_provider/bash_provider.py:24](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/bash_provider/bash_provider.py#L24); searched `rg -n -i 'max_steps|maxSteps|max_iterations|recursion_limit|maxIterations'` in `keep-ui/app keep-ui/features keep/api` → 0 hits (no step cap on the copilot loop in Keep's own code) (verified)
  - *To reach the next level:* Needs an iteration cap plus a wall-clock or token/cost cap enforced in code.
- **C L1:** The timeout covers only the bash provider; other provider calls and the chat loop are unbounded in Keep's code. — [keep/providers/bash_provider/bash_provider.py:24](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/bash_provider/bash_provider.py#L24); [keep-ui/app/api/copilotkit/route.ts:16-22](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep-ui/app/api/copilotkit/route.ts#L16-L22) (verified)
  - *To reach the next level:* Limits should cover the loop and every tool call.
- **D L1:** The timeout default exists but the model can override it, and the rate limiter is off by default. — [keep/providers/bash_provider/bash_provider.py:24](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/providers/bash_provider/bash_provider.py#L24); [keep/api/core/limiter.py:10](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/api/core/limiter.py#L10) (verified)
  - *To reach the next level:* The model shouldn't be able to raise its own limits.
- **B L0:** No ceiling on chat iterations or spend in Keep's code; started subprocesses aren't cancelled by stopping the chat. — [keep/api/core/limiter.py:10](https://github.com/keephq/keep/blob/465e99152e7586fcbe4c2732213d31c5c28def4b/keep/api/core/limiter.py#L10); searched `rg -n -i 'max_steps|maxSteps|max_iterations|recursion_limit|maxIterations'` in `keep-ui/app keep-ui/features keep/api` → 0 hits (no step cap on the copilot loop in Keep's own code) (verified)
  - *To reach the next level:* Needs per-run time and cost ceilings and cancellation of in-flight calls.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: webhook/monitoring alerts (alerts.py:685) passed into chat context (incident-chat.tsx:143) · [B] sensitive data/systems: tenant provider credentials in plaintext state files (filesecretmanager.py:22-29) · [C] state change / egress: invokeProviderMethod to provider methods with no gate (incident-chat.tsx:215-220) · Same default session? Yes

## Highest-impact improvements
1. Restrict /providers/{id}/invoke to an explicit allowlist of provider methods and provider types. — C3 S L0→L2, +0.150 before caps (Playbook 3)
2. Wrap invokeProviderMethod, createIncident and incident updates in renderAndWaitForResponse showing the exact provider, method and parameters. — C2 C L0→L2, +0.150 before caps (Playbook 5)
3. Ship the compose file with DB authentication and a non-admin default role instead of NO_AUTH. — C1 D L0→L2, +0.100 before caps (Playbook 4)
4. Make PostHog and Sentry opt-in and review what authentication-failure logging records. — C8 D L0→L2, +0.100 before caps (Playbook 4)
5. Log invoke arguments and mark AI-initiated actions distinctly from human ones. — C9 S L1→L3, +0.150 before caps (Playbook 1, step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was installed, built or run.
- The commit is 5 commits after tag v0.54.3 and has no tag of its own, so version is null.
- CopilotKit runtime internals (loop limits, stop behaviour) are third-party and weren't examined; C10 credits only Keep's own code.
- The Helm chart lives in a separate repository and wasn't reviewed; the docker-compose defaults were scored.
- The workflow engine was reviewed only where it overlaps the AI paths (bash/python providers); workflow YAML is operator-authored.
- No text aimed at AI reviewers was found in the repository.
