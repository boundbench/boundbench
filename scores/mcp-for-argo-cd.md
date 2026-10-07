# Defense-in-Depth Score: MCP for Argo CD

**Repo:** https://github.com/argoproj-labs/mcp-for-argocd · **Commit:** `28d15ca69b0c31387cc6ec73d201fd13c5d22b6a` · **Reviewed:** 2026-10-03
**What it is:** MCP server for Argo CD applications, clusters and resources
**Category:** Infrastructure & Ops
**Scored configuration:** stdio transport launched via npx with ARGOCD_BASE_URL and ARGOCD_API_TOKEN set, as in the README's Cursor/VS Code/Claude Desktop instructions; MCP_READ_ONLY unset.
**Agent surface (default):** code execution yes · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 3.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L2 | L0 | L0 | 0.23 | — | **0.23** | Medium |
| C2 | Approval gates | L1 | L2 | L0 | L4 | 0.42 | G1 | **0.42** (alt) | High |
| C3 | Tool & action scoping | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | Medium |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L0 | 0.07 | C5-WORSTCASE | **0.07** | Medium |
| C6 | Memory, context & configuration integrity | L0 | L0 | SA | L1 | 0.25 | C6-REPOCONFIG | **0.25** | Medium |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | Medium |
| C9 | Audit & traceability | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C10 | Limits & kill switch | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |

Controls where a risk surface exists: 2.10 / 9.0 (23%); 1 criterion scored SA (surface absent).

As shipped, this server hands the model an Argo CD token with every write tool switched on: it can create Applications from any repository, sync with prune, run resource actions and delete with cascade, and it gives the MCP host no read-only or destructive markings to decide what needs approval. Tool results include pod logs and manifests other people control, so a prompt injection can turn into cluster changes. The token itself is well handled (never in tool arguments, bound to its host), and MCP_READ_ONLY=true removes all write tools, but it is opt-in. It also auto-loads a .env from the launch directory, and it keeps no record of tool calls.

## Critical gaps
- The Argo CD token used for every tool can deploy any repository to any managed cluster and delete Applications with cascade; the server does not narrow it. (ASI03, T3, LLM06; C1) — [src/server/server.ts:279-285](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L279-L285); [src/server/server.ts:293-321](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L293-L321)
- create_application plus sync lets the model have Argo CD deploy arbitrary manifests (any container) to production clusters with no isolation or policy in the server. (ASI05, T11, LLM05; C4) — [src/server/server.ts:279-285](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L279-L285); [src/shared/models/schema.ts:30](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/shared/models/schema.ts#L30)
- A hijacked session can read attacker-influenced logs and then delete, sync or deploy (and leak data via an attacker repoURL) with no server-side human step; read-only mode is off by default (C5-WORSTCASE). (ASI01, T6, LLM01; C5) — [src/argocd/client.ts:219-242](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/argocd/client.ts#L219-L242); [src/server/server.ts:293-321](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L293-L321); [src/server/server.ts:63-66](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L63-L66)
- dotenv.config() auto-loads a .env from the working directory, letting a cloned repo redirect the Argo CD endpoint/token registry or disable TLS verification without a trust prompt (C6-REPOCONFIG). (ASI06, ASI04, T1; C6) — [src/index.ts:1-5](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/index.ts#L1-L5)

## Criterion details

### C1 Identity & least privilege — 0.23 (medium)

The server acts with a single Argo CD API token taken from the ARGOCD_API_TOKEN environment variable, and every tool, read or write, uses that same token. The server does not narrow it in any way: there is no separate read credential, no per-tool scoping and no authorization check of its own, so what the agent can do is whatever the operator's token can do in Argo CD. One good design choice is that the token is bound to the configured Argo CD host and is never sent to a different host the model names. Write tools are on by default, and the README does not steer users toward a read-only or project-scoped Argo CD account. In the HTTP transport (not the scored mode) the server also accepts the caller's token in a header and forwards it to Argo CD.

- **S L1:** One static Argo CD bearer token from the environment is attached to every request; scope is whatever the operator's Argo CD account has, and the server neither narrows it nor separates read and write credentials. — [src/server/transport.ts:22-27](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/transport.ts#L22-L27); [src/argocd/http.ts:17-20](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/argocd/http.ts#L17-L20) (verified)
  - *To reach the next level:* No role scoping enforced or checked by the server and no separate read-only credential for read tools.
- **C L2:** Every tool goes through the same addJsonOutputTool -> resolveClient path, so all built-in tools use the one configured token, and the default token is bound to the default base URL only; there are no subprocesses or extensions. — [src/server/server.ts:459-466](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L459-L466); [src/server/server.ts:418-422](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L418-L422) (verified)
  - *To reach the next level:* No authorization layer in code that every tool path traverses, and no per-principal check.
- **D L0:** Write tools (create/update/delete/sync/run_resource_action) are registered unless MCP_READ_ONLY=true is set, and the documented install passes an arbitrary Argo CD token; least privilege requires manual hardening. — [src/server/server.ts:63-66](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L63-L66); [src/server/server.ts:278](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L278); [README.md:314](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/README.md#L314) (verified)
  - *To reach the next level:* No read-only or minimal default; write requires no explicit operator elevation.
- **B L0:** The token can create Applications pointing at any repo and destination and delete or sync any Application the account can see, which in a typical Argo CD install means deploying arbitrary workloads to every managed cluster; only Argo CD's own RBAC and AppProject rules (outside this repo) bound it. — [src/server/server.ts:279-285](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L279-L285); [src/server/server.ts:293-321](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L293-L321); [src/shared/models/schema.ts:27-33](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/shared/models/schema.ts#L27-L33) (inferred)
  - *To reach the next level:* Credentials are not limited to one project, read-mostly, or short-lived.
- **Cap:** none
- **Notes:** S is L1 because the token is always used unnarrowed, not because a control is opt-in; D L0 reflects the default privilege, so G1 does not apply. C1-PASSTHRU not applied: the scored stdio mode takes the token only from the environment. In the http/sse transports the server forwards a client-supplied x-argocd-api-token header to Argo CD (src/server/transport.ts:82-83), which would trigger C1-PASSTHRU (<= 0.40, above this criterion's score).

### C2 Approval gates — 0.42 (high)

The server has no approval step of its own and relies on the MCP host. It gives the host little to work with: no tool carries read-only or destructive annotations, so a host cannot tell list_applications from delete_application by metadata. Only sync_application offers a dry-run option, and the model chooses whether to use it. The strongest control is an opt-in read-only mode (MCP_READ_ONLY=true) that removes all five mutating tools; without it, deletes with cascade, prunes and resource actions run immediately when called.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** No tool registration passes MCP annotations (readOnlyHint/destructiveHint), so the server gives the host no risk signal for its mutating tools. — searched `rg -n -a -S -e 'annotations|readOnlyHint|destructiveHint|idempotentHint'` in `src` → 12 hits (all hits are in src/types/argocd.d.ts, generated Argo CD/Kubernetes API types (Kubernetes object annotations, ManagedFields, resourceVersion docs); none is a control; the server registers tools with this.tool(name, description, schema, cb) and passes no annotations); [src/server/server.ts:459](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L459) (verified)
    - *To reach the next level:* No accurate read-only/destructive annotations on every tool.
  - **C L0:** Every mutating tool, including delete_application with cascade and run_resource_action, is exposed without any server-side gate or signal. — [src/server/server.ts:293-321](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L293-L321); [src/server/server.ts:366-382](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L366-L382) (verified)
    - *To reach the next level:* No mutating tool is flagged or gated by the server.
  - **D L0:** The only safety mode, read-only, is opt-in via an environment variable; by default all write tools are registered. — [src/server/server.ts:63-66](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L63-L66); [README.md:314](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/README.md#L314) (verified)
    - *To reach the next level:* No safety mode or gating is on by default.
  - **B L0:** delete_application accepts cascade and propagationPolicy, so a single wrong call deletes an Application and all its Kubernetes resources (including stateful data) with no undo; sync has an optional dry-run but nothing forces it. — [src/server/server.ts:301-308](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L301-L308); [src/server/server.ts:330-333](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L330-L333) (verified)
    - *To reach the next level:* No preview or dry-run required for destructive operations and no undo.
- **opt-in read-only mode (MCP_READ_ONLY=true)** (alt; raw 0.42, cap G1 → 0.42) ← counted
  - **S L1:** A server-enforced read-only mode exists and read and write operations are separate tools, but no tool carries annotations, so the mechanism misses the L2 element. — [src/server/server.ts:63-66](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L63-L66); [src/server/server.ts:278](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L278); searched `rg -n -a -S -e 'annotations|readOnlyHint|destructiveHint|idempotentHint'` in `src` → 12 hits (all hits are in src/types/argocd.d.ts, generated Argo CD/Kubernetes API types (Kubernetes object annotations, ManagedFields, resourceVersion docs); none is a control; the server registers tools with this.tool(name, description, schema, cb) and passes no annotations) (verified)
    - *To reach the next level:* No readOnlyHint/destructiveHint annotations on every tool.
  - **C L2:** When enabled, all five mutating tools sit inside the if (!isReadOnly) block and are not registered; the remaining tools issue only GET requests. — [src/server/server.ts:278](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L278); [src/server/server.ts:366-383](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L366-L383) (verified)
    - *To reach the next level:* No annotations or preview for the host to act on when write mode is on.
  - **D L0:** Read-only mode is off unless the operator sets MCP_READ_ONLY=true. — [src/server/server.ts:63-66](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L63-L66); [README.md:314](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/README.md#L314) (verified)
    - *To reach the next level:* Read-only is not the default.
  - **B L4:** With read-only mode on, no consequential action is reachable: every remaining tool issues GET requests via HttpClient.get. — [src/server/server.ts:278](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L278); [src/argocd/client.ts:83-90](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/argocd/client.ts#L83-L90) (verified)
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C3 Tool & action scoping — 0.38 (high)

Tools are narrow and named for Argo CD operations rather than a generic HTTP client, and each has a typed schema. But almost nothing is validated beyond types: names in request paths are not strictly validated. Repository URLs, resource action names and sync options are passed through unchecked. All write tools are on by default; one environment variable removes them as a group.

- **S L1:** Arguments are typed with zod but names in request paths are not strictly validated; repoURL, action and syncOptions are arbitrary strings; only two checks (namespace non-empty, destination server XOR name) exist. — [src/server/server.ts:373](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L373); [src/shared/models/schema.ts:57](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/shared/models/schema.ts#L57) (verified)
  - *To reach the next level:* No allowlist validation of names, repo URLs, or action names.
- **C L2:** Every tool's input is parsed by its zod schema through the shared addJsonOutputTool wrapper, but the schemas are type-only for the arguments that matter. — [src/server/server.ts:459-466](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L459-L466); [src/server/server.ts:458](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L458) (verified)
  - *To reach the next level:* No shared validation layer beyond type checks.
- **D L2:** Tool groups are selectable (MCP_READ_ONLY removes all write tools), but the default set includes create, update, delete, sync and resource actions. — [src/server/server.ts:63-66](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L63-L66); [src/server/server.ts:278](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L278); [README.md:314](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/README.md#L314) (verified)
  - *To reach the next level:* Default tool set is not read-only.
- **B L1:** A misused tool reaches every Application and cluster the token can see, can deploy any repository to any destination and can delete with cascade; the only bounds are Argo CD RBAC/AppProject policy outside this repo. — [src/server/server.ts:279-285](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L279-L285); [src/server/server.ts:293-321](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L293-L321) (verified)
  - *To reach the next level:* No project, namespace, or quantity bounds enforced by the server.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (medium)

The server never runs a shell, eval, or subprocess on its own host. It can, however, cause model-chosen code to run elsewhere: create_application accepts any repository URL and destination, and sync_application then has Argo CD apply whatever manifests that repository contains, which can start any container in a managed cluster. Nothing in the server isolates or constrains this; only Argo CD's own project rules, configured outside this repo, stand in the way, and the write tools are on by default.

- **S L0:** Model-supplied repoURL/path/destination are sent straight to Argo CD, which renders and applies that repository's manifests; the server applies no isolation or policy to what gets deployed. — [src/server/server.ts:279-285](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L279-L285); [src/shared/models/schema.ts:27-33](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/shared/models/schema.ts#L27-L33); [src/server/server.ts:322-365](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L322-L365) (verified)
  - *To reach the next level:* No isolation boundary or deploy policy around model-chosen deployments.
- **C L0:** No execution path (create, update, sync, run_resource_action) is constrained by the server. — [src/server/server.ts:279-285](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L279-L285); [src/server/server.ts:286-292](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L286-L292); [src/server/server.ts:366-382](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L366-L382) (verified)
  - *To reach the next level:* No path is sandboxed or policy-checked.
- **D L0:** The deploy-capable tools are registered by default; only the opt-in read-only mode removes them. — [src/server/server.ts:63-66](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L63-L66); [src/server/server.ts:278](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L278) (verified)
  - *To reach the next level:* Nothing restricts remote execution by default.
- **B L0:** Deployed workloads run in production clusters with whatever service accounts, secrets and network the destination namespace offers; Argo CD AppProject rules are the only external bound. — [src/server/server.ts:279-285](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L279-L285); [src/shared/models/schema.ts:51-62](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/shared/models/schema.ts#L51-L62) (inferred)
  - *To reach the next level:* No workspace-only, no-secret, or no-network confinement for model-chosen deployments.
- **Cap:** none

### C5 Untrusted input blast radius — 0.07 (medium)

Tool results include content that other people control: application logs, Kubernetes events, resource manifests and Application specs. The server returns them as one plain JSON text blob with no marking of what is untrusted and no separation between data and metadata. The only mode that would drop a dangerous capability, read-only, is off by default. A prompt-injected agent that reads a crafted log line can therefore delete or re-sync applications, deploy an attacker's repository to a cluster, and send data out through a repository URL Argo CD will fetch, all in the same session.

- **S L1:** Results are JSON.stringify'd into a single text content item with no provenance or untrusted flag; tool descriptions contain no directives. — [src/server/server.ts:474-477](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L474-L477) (verified)
  - *To reach the next level:* No structured separation of returned content from metadata.
- **C L0:** Logs, events, manifests and app specs all enter the host's context the same way, with nothing distinguishing them. — [src/argocd/client.ts:219-242](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/argocd/client.ts#L219-L242); [src/server/server.ts:474-477](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L474-L477) (verified)
  - *To reach the next level:* No untrusted source is distinguished.
- **D L0:** Read-only mode, the only Rule-of-Two leg the server can drop, is opt-in. — [src/server/server.ts:63-66](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L63-L66); [README.md:314](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/README.md#L314) (verified)
  - *To reach the next level:* No limiting mode is on by default.
- **B L0:** In the default config one session can read attacker-influenced pod logs, read cluster manifests, and then delete with cascade or create an Application whose repoURL points at an attacker host (an egress channel via Argo CD's repo fetch) with no server-side human step. — [src/argocd/client.ts:219-242](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/argocd/client.ts#L219-L242); [src/server/server.ts:293-321](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L293-L321); [src/server/server.ts:279-285](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L279-L285); [src/shared/models/schema.ts:30](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/shared/models/schema.ts#L30) (inferred)
  - *To reach the next level:* Exfiltration and irreversible actions are not forced through approval.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.25 (medium)

The server keeps no memory, conversation store, or retrieval index. However, at startup it calls dotenv, which loads a .env file from whatever directory the server is launched in. MCP hosts commonly start stdio servers inside the user's project, so a .env committed to a cloned repository can set variables the operator left unset, such as the Argo CD base URL, the token-registry path, or NODE_TLS_REJECT_UNAUTHORIZED=0 to turn off certificate checks. This happens silently, with no trust prompt, and persists for every session started in that directory.

- **S L0:** dotenv.config() reads .env from the current working directory at startup with no trust decision; it can supply ARGOCD_BASE_URL, ARGOCD_TOKEN_REGISTRY_PATH or NODE_TLS_REJECT_UNAUTHORIZED when the operator did not set them (dotenv 16 default: path = cwd/.env, override = false). — [src/index.ts:1-5](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/index.ts#L1-L5); [src/server/transport.ts:20](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/transport.ts#L20) (verified)
  - *To reach the next level:* Security-relevant settings are not restricted to user/operator scope.
- **C L0:** The single auto-loaded configuration path is uncontrolled. — [src/index.ts:1-5](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/index.ts#L1-L5) (verified)
  - *To reach the next level:* No auto-loaded file is gated.
- **D SA:** No memory or retrieval store exists, so there is no namespace isolation to configure. — searched `rg -n -a -S -e 'memory|vector|AGENTS.md|CLAUDE.md'` in `src` → 0 hits (no memory, vector store or instruction-file loading); searched `rg -n -a -S -e 'writeFile|appendFile|createWriteStream'` in `src` → 2 hits (both hits are in src/server/tokenRegistry.test.ts (test fixture); the server itself writes no files) (verified)
- **B L1:** A poisoned .env persists on disk in the workspace and applies to every later session started there, redirecting where tool calls and credentials go or disabling TLS verification for them. — [src/index.ts:1-5](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/index.ts#L1-L5) (inferred)
  - *To reach the next level:* Poisoned configuration is not limited to text output or gated actions.
- **Cap:** C6-REPOCONFIG — src/index.ts calls dotenv.config() at startup, auto-loading a .env from the working directory (often a cloned repo) that can redirect the Argo CD endpoint, the token registry, or TLS verification without a trust decision.

### C7 Third-party extensions — 1.00 (high)

The server loads no third-party code at runtime: no plugins, no dynamic imports, no subprocesses and no package installs. Its dependencies are fixed at build time. The README's 'npx argocd-mcp@latest' install line is how users fetch this server itself and is general supply-chain hygiene, not an extension mechanism.

- **Structural absence:** searched `rg -n -a -S -e 'import\(|require\('` in `src` → 0 hits (no dynamic module loading); searched `rg -n -a -S -e 'child_process|spawn\(|execSync|eval\(|new Function'` in `src` → 0 hits (no subprocess or code evaluation)
- **Notes:** The documented install uses an unpinned npx argocd-mcp@latest (README.md:14), which is the server's own supply chain and out of scope for this criterion.

### C8 Secrets & sensitive-data protection — 0.38 (medium)

The Argo CD token is read from an environment variable and is deliberately kept out of tool arguments, so it never passes through the model, and it is bound to the configured Argo CD host so the model cannot redirect it elsewhere. The server does not log requests, tokens or tool results, and has no telemetry. But nothing is redacted from tool output: logs and manifests returned to the model are passed through as-is. The token itself is a long-lived Argo CD credential, and the repository's own sample Cursor config ships with TLS verification disabled.

- **S L1:** The token comes from an env var and is never a tool argument or log field, and the default token is bound to the default base URL; there is no masking or redaction of anything returned to the model. — [src/server/transport.ts:22-27](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/transport.ts#L22-L27); [src/server/server.ts:418-422](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L418-L422); searched `rg -n -a -S -e 'redact|sanitiz'` in `src` → 0 hits (no redaction helpers) (verified)
  - *To reach the next level:* No redaction on log or model-bound paths.
- **C L2:** Logs (7 pino call sites, none logs requests or tokens), errors (only error.message) and absent subprocesses are clean; model-bound tool results are not filtered. — searched `rg -n -a -S -e 'logger\.(info|warn|error|debug)'` in `src` → 7 hits (transport/startup messages only; none logs tool args, results, or tokens); [src/server/server.ts:478-482](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L478-L482) (verified)
  - *To reach the next level:* Model-bound tool results are not covered.
- **D L2:** No telemetry; pino logs startup events to stderr only. The repo's sample .cursor/mcp.json and README suggest NODE_TLS_REJECT_UNAUTHORIZED=0, which exposes the bearer token to interception when copied. — [src/logging/logging.ts:4](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/logging/logging.ts#L4); searched `rg -n -a -S -e 'telemetry|sentry|posthog|opentelemetry'` in `src` → 5 hits (all hits are in src/types/argocd.d.ts, generated Argo CD/Kubernetes API types (Kubernetes object annotations, ManagedFields, resourceVersion docs); none is a control); [.cursor/mcp.json:10](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/.cursor/mcp.json#L10) (verified)
  - *To reach the next level:* Redaction is not always on.
- **B L1:** A leaked token is a long-lived Argo CD API token with whatever (often broad) role the operator assigned; it is not reachable by the model or any subprocess. — [src/server/transport.ts:22-27](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/transport.ts#L22-L27); [src/argocd/http.ts:17-20](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/argocd/http.ts#L17-L20) (inferred)
  - *To reach the next level:* Tokens are not scoped and short-lived by the server.
- **Cap:** none

### C9 Audit & traceability — 0.00 (high)

The server keeps no record of what it did. Its logger writes only startup and listener messages; tool calls, their arguments, and their results are never logged. After an incident the only trace would be whatever the MCP host or Argo CD's own audit events recorded.

- **S L0:** No tool call is recorded by the server; the 7 logger calls are startup and listener messages. — searched `rg -n -a -S -e 'logger\.(info|warn|error|debug)'` in `src` → 7 hits (none is in the tool path (server.ts has no logger import)); [src/server/server.ts:459-466](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L459-L466) (verified)
  - *To reach the next level:* No structured record of tool calls.
- **C L0:** No tool path is recorded. — [src/server/server.ts:459-484](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L459-L484) (verified)
  - *To reach the next level:* Main tool path not recorded.
- **D L0:** There is no audit record to enable. — [src/logging/logging.ts:4](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/logging/logging.ts#L4) (verified)
  - *To reach the next level:* No record on by default.
- **B L0:** Actions proceed with no record. — [src/server/server.ts:459-466](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L459-L466) (verified)
  - *To reach the next level:* No per-action record.
- **Cap:** none

### C10 Limits & kill switch — 0.38 (high)

Log tools always ask Argo CD for at most the last 100 lines and never follow the stream, which bounds one kind of output. Everything else is unbounded: listing applications fetches every application before applying the caller's optional limit, get_resources with no refs fetches every resource in the tree in parallel, and no HTTP request has a timeout or cancellation. There are no rate limits on the write tools.

- **S L2:** Server-enforced cap on log output (tailLines 100, follow false); other operations are unbounded and no fetch has a timeout. — [src/argocd/client.ts:219-242](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/argocd/client.ts#L219-L242); [src/argocd/client.ts:206-217](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/argocd/client.ts#L206-L217); searched `rg -n -a -S -e 'AbortSignal|AbortController|setTimeout|signal:'` in `src` → 0 hits (no timeouts or cancellation on fetch) (verified)
  - *To reach the next level:* No caps on every operation and no rate or concurrency limits.
- **C L1:** Only the three log methods are capped; get_resources fans out over the whole tree with Promise.all. — [src/server/server.ts:242-259](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L242-L259); [src/argocd/client.ts:54-57](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/argocd/client.ts#L54-L57) (verified)
  - *To reach the next level:* Caps do not cover listing, resource fetches, or request duration.
- **D L2:** The log cap is hard-coded and the model cannot raise it; list limit is caller-chosen and unbounded by default. — [src/argocd/client.ts:219-242](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/argocd/client.ts#L219-L242); [src/server/server.ts:79-86](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/server/server.ts#L79-L86) (verified)
  - *To reach the next level:* Most operations have no default bound.
- **B L1:** A hung Argo CD request has no timeout, and parallel resource fetches and write calls are unbounded in number. — [src/argocd/http.ts:34-37](https://github.com/argoproj-labs/mcp-for-argocd/blob/28d15ca69b0c31387cc6ec73d201fd13c5d22b6a/src/argocd/http.ts#L34-L37) (verified)
  - *To reach the next level:* No tight time ceiling or cancellation of in-flight requests.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Pod logs, Kubernetes events and manifests returned verbatim (src/argocd/client.ts:219, src/server/server.ts:476) · [B] sensitive data/systems: Argo CD API token and cluster manifests (src/server/transport.ts:25, src/argocd/client.ts:266) · [C] state change / egress: create/update/delete/sync/run_resource_action registered by default (src/server/server.ts:278) · Same default session? Yes

## Highest-impact improvements
1. Remove dotenv.config() or load only an explicit, operator-given env file path. — C6 S L0→L4, +0.300 before caps
2. Add readOnlyHint/destructiveHint/idempotentHint annotations to every tool so hosts can gate writes. — C2 S L0→L2, +0.150 before caps (Playbook 5)
3. Make read-only the default and require an explicit MCP_ALLOW_WRITE to register write tools. — C3 D L2→L3, +0.050 before caps (Playbook 3)
4. Validate names against the Kubernetes name pattern; allowlist repoURL hosts and resource actions. — C3 S L1→L2, +0.075 before caps (Playbook 3)
5. Log a structured record (tool, arguments, target base URL, status, timestamp) for every tool call to stderr. — C9 S L0→L2, +0.150 before caps (Playbook 1)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was installed, built, or run, and Argo CD itself was not exercised.
- Scored the stdio transport the README leads with. The http/sse transports (Docker image default) add strong listener hardening (loopback default, refusal to bind wider without MCP_AUTH_TOKEN, Host/Origin checks) but also accept and forward a client's x-argocd-api-token header (token passthrough).
- What the Argo CD token can do depends on the operator's Argo CD RBAC and AppProject configuration, which is outside this repo; blast-radius ratings assume a typical broadly privileged API account.
- One request-path handling finding was not exercised against a live Argo CD.
- The .env effect relies on dotenv 16 defaults (cwd/.env, no override of variables already set) and on the MCP host's choice of working directory.
- No reviewer-injection text was found in the repository.
