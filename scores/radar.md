# Defense-in-Depth Score: Radar

**Repo:** https://github.com/skyhook-io/radar · **Commit:** `09719d88ba15c66b5b772b4c483b028c3c2f55ab` · **Reviewed:** 2026-10-03
**What it is:** Open-source Kubernetes UI with built-in MCP server for AI agents (restart/scale/apply/rollback/exec)
**Category:** Infrastructure & Ops
**Scored configuration:** Local binary (`kubectl radar`) with no flags: auth-mode none, 127.0.0.1:9280, MCP enabled, full read+write /mcp endpoint using the user's kubeconfig identity.
**Agent surface (default):** code execution yes · filesystem write no · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication no

## Score: 3.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L3 | L3 | L2 | L0 | 0.55 | G1 | **0.50** (alt) | High |
| C2 | Approval gates | L2 | L3 | L2 | L1 | 0.53 | — | **0.53** | High |
| C3 | Tool & action scoping | L1 | L2 | L2 | L0 | 0.33 | — | **0.33** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L2 | L2 | L2 | L0 | 0.40 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L2 | L2 | L2 | L3 | 0.55 | — | **0.55** | High |
| C7 | Third-party extensions | L1 | L1 | L2 | L2 | 0.35 | — | **0.35** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |


Radar's MCP server gives an AI agent the full authority of your kubeconfig, with no authentication on the local /mcp endpoint and write tools on by default, including apply_resource, which creates any Kubernetes object, privileged pods among them. Risk labelling for agent hosts is good (accurate read-only and destructive hints, dry-run for apply and patch, a separate read-only endpoint), and Secret values are kept out of model context. The dominant risk is that a prompt-injected agent reading pod logs or annotations can deploy arbitrary workloads with cluster-admin reach. Point agents at /mcp-readonly unless you need writes.

## Critical gaps
- In default local mode MCP tools act with the operator's full kubeconfig identity, typically cluster-admin, over an unauthenticated localhost endpoint. (ASI03; C1) — [internal/k8s/context_client.go:47-61](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/k8s/context_client.go#L47-L61); [cmd/explorer/main.go:163](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/cmd/explorer/main.go#L163)
- apply_resource applies model-written manifests of any kind with no pod-security or kind policy, so the model chooses privileged/host-mounted workloads in the cluster. (ASI05; C4) — [pkg/k8score/workload.go:286-321](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/pkg/k8score/workload.go#L286-L321); searched `rg -n -i 'privileged|hostPath|hostNetwork|PodSecurity|allowedKinds|kindAllow|denyKind'` in `internal/mcp/tools_apply.go internal/mcp/tools_patch.go pkg/k8score/workload.go` → 0 hits (no kind allowlist or pod-security check on the apply/patch path)
- The default /mcp session combines untrusted cluster content, Secret-reachable cluster access and unrestricted write tools, so a hijacked agent can leak data and make irreversible changes unattended (C5-WORSTCASE). (ASI01, LLM01; C5) — [internal/app/bootstrap.go:566-567](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/app/bootstrap.go#L566-L567); [pkg/k8score/workload.go:286-321](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/pkg/k8score/workload.go#L286-L321)

## Criterion details

### C1 Identity & least privilege — 0.50 (high)

In the default local mode Radar's MCP server acts with whatever identity your kubeconfig holds, often cluster-admin, and the /mcp endpoint has no authentication of its own, so any agent (or local process) that reaches it gets that full authority, including write tools. Radar does not narrow the credential or check each request against a least-privilege policy; Kubernetes RBAC on your own account is the only bound. An optional in-cluster deployment with proxy or OIDC authentication runs every write as the signed-in user through Kubernetes impersonation and filters cached reads per user, which is a real per-request authorization layer, but it is off by default and its ServiceAccount holds the cluster-wide impersonate privilege.

- **default configuration** (default; raw 0.07 → 0.07)
  - **S L0:** With auth-mode none the MCP tools use the shared kubeconfig/in-cluster client directly (ambient operator authority, no narrowing). — [internal/k8s/context_client.go:47-61](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/k8s/context_client.go#L47-L61); [cmd/explorer/main.go:163](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/cmd/explorer/main.go#L163) (verified)
    - *To reach the next level:* Use a dedicated, role-scoped identity or a deterministic least-privilege gate before credentials are attached.
  - **C L1:** Every MCP tool obtains its client through the same ClientFromContext/DynamicClientFromContext helper, but in default mode that helper returns the ambient client with no authorization check. — [internal/mcp/tools_apply.go:56](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools_apply.go#L56); [internal/k8s/context_client.go:47-61](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/k8s/context_client.go#L47-L61) (verified)
    - *To reach the next level:* Route every tool through an authorization layer that checks the request, not only a shared client constructor.
  - **D L0:** Default is auth-mode none on a local binary using the user's kubeconfig; the full read+write /mcp handler is mounted by default and documented as the endpoint to configure. — [cmd/explorer/main.go:163](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/cmd/explorer/main.go#L163); [internal/app/bootstrap.go:566-567](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/app/bootstrap.go#L566-L567); [docs/mcp.md:81](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/docs/mcp.md#L81) (verified)
    - *To reach the next level:* Default to a read-only or minimal identity and require explicit elevation for writes.
  - **B L0:** Write tools include apply_resource of arbitrary kinds and node drain; with a typical admin kubeconfig a hijacked caller has cluster-admin-equivalent reach over production clusters. — [internal/mcp/tools.go:593-610](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools.go#L593-L610); [pkg/k8score/workload.go:286-321](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/pkg/k8score/workload.go#L286-L321) (verified)
    - *To reach the next level:* Bound the identity so a failure reaches one namespace or read-only data.
- **opt-in auth mode (proxy/OIDC) with per-user impersonation** (alt; raw 0.55, cap G1 → 0.50) ← counted
  - **S L3:** With auth enabled, write clients are built per request by impersonating the authenticated user, so Kubernetes RBAC evaluates each call against the requesting principal. — [internal/k8s/context_client.go:47-55](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/k8s/context_client.go#L47-L55); [pkg/auth/impersonate.go:11-18](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/pkg/auth/impersonate.go#L11-L18) (verified)
    - *To reach the next level:* Narrow the credential itself (short-lived, intersected with an agent policy) rather than relying on impersonation of the user's full rights.
  - **C L3:** Cache-served reads are filtered per user via SubjectAccessReview-based namespace discovery and fail closed when discovery cannot run; writes, exec and logs are impersonated. — [internal/mcp/permissions.go:110-122](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/permissions.go#L110-L122); [internal/mcp/permissions.go:39-51](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/permissions.go#L39-L51) (verified)
    - *To reach the next level:* Make policy resolution failure deny on every path.
  - **D L2:** Auth is opt-in (default none); once on, the user's own RBAC tier decides writes with no read-only default for agents. — [cmd/explorer/main.go:163](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/cmd/explorer/main.go#L163) (verified)
    - *To reach the next level:* Make the agent-facing endpoint read-only by default under auth and require elevation for writes.
  - **B L0:** The chart grants the Radar ServiceAccount impersonate on all users and groups under auth, and proxy mode trusts a username header; if that layer is bypassed the SA can act as any identity. — [deploy/helm/radar/templates/clusterrole.yaml:546-552](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/deploy/helm/radar/templates/clusterrole.yaml#L546-L552); [internal/auth/middleware.go:114](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/auth/middleware.go#L114) (verified)
    - *To reach the next level:* Restrict impersonation to specific groups/users (resourceNames) so a bypass cannot reach admin identities.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.
- **Notes:** Apply can create RoleBindings; Kubernetes escalation prevention limits this to permissions the identity already holds, so C1-SELFESC was not applied. Any local process (not only the intended agent) can call /mcp; the SDK's DNS-rebinding Host check and http.CrossOriginProtection block browser-originated calls (internal/mcp/server.go:139-188).

### C2 Approval gates — 0.53 (high)

Radar is a tool server, so the agent host decides what to confirm; Radar's job is to label risk accurately. It does that well: read and write tools are separate, every read tool carries readOnlyHint, every write tool carries destructiveHint, and a separate /mcp-readonly endpoint drops write tools entirely. apply_resource, patch_resource and Argo sync/rollback support server-side dry-run previews, but restart, scale, rollback, cordon/drain and CronJob actions have no preview. The default endpoint that all setup instructions use exposes the write tools, and nothing on the server side requires a confirmation step.

- **S L2:** Separate read/write tools with readOnlyHint on reads, destructiveHint on all seven write tools, and diagnose(in_cluster) honestly marked neither. — [internal/mcp/tools.go:60-63](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools.go#L60-L63); [internal/mcp/tools.go:69-72](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools.go#L69-L72); [internal/mcp/tools.go:79-82](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools.go#L79-L82); [internal/mcp/tools_workloads.go:721-727](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools_workloads.go#L721-L727) (verified)
  - *To reach the next level:* Offer a dry-run/preview for every destructive operation (manage_workload, manage_node drain, manage_cronjob, manage_rollout have none).
- **C L3:** All 34 tool registrations carry an annotation; write tools are registered only on the full server so the read-only mount cannot even list them. — [internal/mcp/tools.go:542-544](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools.go#L542-L544); searched `rg -n 'Annotations: writeTool'` in `internal/mcp/tools.go` → 7 hits (one per write tool) (verified)
  - *To reach the next level:* Add a server-enforced confirmation step or read-only default so unlabeled host behaviour cannot reach writes.
- **D L2:** Annotations are hard-coded and cannot be disabled, but the default /mcp mount (and every documented client snippet) exposes write tools; read-only requires pointing at a different URL. — [internal/app/bootstrap.go:566-567](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/app/bootstrap.go#L566-L567); [docs/mcp.md:81](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/docs/mcp.md#L81) (verified)
  - *To reach the next level:* Default agent connections to /mcp-readonly and require an explicit operator flag for the write endpoint.
- **B L1:** Writes reach arbitrary manifests, node drain, Argo sync with prune (default true) and terminate; there is no undo beyond Kubernetes rollout history and no rate limit on write calls. — [internal/mcp/tools_gitops.go:25](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools_gitops.go#L25); searched `rg -n -i 'rate\.NewLimiter|ratelimit'` in `internal/mcp` → 0 hits (no rate limiting on MCP tool calls) (verified)
  - *To reach the next level:* Provide previews for all write actions and bound quantities (replica ceilings, rate limits on mutations).
- **Cap:** none
- **Notes:** Radar's in-app investigation runner only enables write tools in a separate session bound to user-confirmed fix text (internal/server/ai_diagnose.go:448-455); this is product UI, not the MCP server's own gate, and is not credited here.

### C3 Tool & action scoping — 0.33 (high)

Most tools take typed, closed schemas (additionalProperties false) with enumerated actions, and several reads clamp their limits. But the two most powerful tools are general-purpose: apply_resource accepts any multi-document YAML of any kind and patch_resource accepts any JSON/merge/strategic patch, so a misused agent can create RBAC bindings, privileged pods or anything else its identity allows. Scale has no replica ceiling and drain timeout is caller-chosen. The default endpoint ships the write group; a read-only group exists but must be chosen.

- **S L1:** Typed schemas with enumerated actions, but apply_resource/patch_resource pass arbitrary manifests and patches to any discovered GVR with no kind allowlist or field policy. — [pkg/k8score/workload.go:286-321](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/pkg/k8score/workload.go#L286-L321); searched `rg -n -i 'privileged|hostPath|hostNetwork|PodSecurity|allowedKinds|kindAllow|denyKind'` in `internal/mcp/tools_apply.go internal/mcp/tools_patch.go pkg/k8score/workload.go` → 0 hits (no kind allowlist or pod-security check on the apply/patch path) (verified)
  - *To reach the next level:* Replace generic apply/patch with validated, narrow operations or enforce a kind/field allowlist in code.
- **C L2:** Most tools validate enumerated actions and some clamp limits (events <=100, changes <=50), but scale replicas and drain timeout are unbounded. — [internal/mcp/tools.go:1947-1948](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools.go#L1947-L1948); [internal/mcp/tools_workloads.go:121-127](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools_workloads.go#L121-L127); [internal/mcp/tools_workloads.go:721-727](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools_workloads.go#L721-L727) (verified)
  - *To reach the next level:* Apply a shared validation layer with numeric bounds to every tool.
- **D L2:** Tool groups are selectable (full vs read-only mount) but the default/documented group includes all write tools. — [internal/mcp/tools.go:542-544](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools.go#L542-L544); [internal/app/bootstrap.go:566-567](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/app/bootstrap.go#L566-L567) (verified)
  - *To reach the next level:* Ship the read-only tool set by default and require explicit enabling of writes.
- **B L0:** apply_resource can create any resource in any namespace the operator's kubeconfig reaches, i.e. a general-purpose tool against production clusters. — [internal/mcp/tools.go:593-610](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools.go#L593-L610); [internal/mcp/tools_apply.go:56](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools_apply.go#L56) (verified)
  - *To reach the next level:* Scope tools to a namespace/project and bound quantities.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

Radar never runs model code on your laptop, but apply_resource turns model-written YAML into running workloads in your cluster: the model chooses the image, command and security context, including privileged mode, host networking and host path mounts. Radar applies no pod-security or kind policy of its own, so whatever isolation exists comes from the cluster's admission controls, not from Radar. Combined with the operator's ambient credentials, a hijacked agent can run arbitrary code with node-level reach.

- **S L0:** Model-authored manifests are applied verbatim; the model defines the container's isolation (securityContext, hostPath, hostNetwork) so there is no boundary it cannot redefine. — [pkg/k8score/workload.go:286-321](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/pkg/k8score/workload.go#L286-L321); searched `rg -n -i 'privileged|hostPath|hostNetwork|PodSecurity|allowedKinds|kindAllow|denyKind'` in `internal/mcp/tools_apply.go internal/mcp/tools_patch.go pkg/k8score/workload.go` → 0 hits (no kind allowlist or pod-security check on the apply/patch path) (verified)
  - *To reach the next level:* Enforce a fixed pod-security profile (non-root, no privilege, no host namespaces/mounts) on anything the MCP path creates.
- **C L0:** Neither apply_resource nor patch_resource (which can rewrite pod templates) passes through any isolation policy. — searched `rg -n -i 'privileged|hostPath|hostNetwork|PodSecurity|allowedKinds|kindAllow|denyKind'` in `internal/mcp/tools_apply.go internal/mcp/tools_patch.go pkg/k8score/workload.go` → 0 hits (no kind allowlist or pod-security check on the apply/patch path); [internal/mcp/tools_patch.go:21-30](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools_patch.go#L21-L30) (verified)
  - *To reach the next level:* Route every workload-creating path (apply, patch, CronJob trigger) through the same enforced policy.
- **D L0:** No isolation control exists to be on by default. — searched `rg -n -i 'privileged|hostPath|hostNetwork|PodSecurity|allowedKinds|kindAllow|denyKind'` in `internal/mcp/tools_apply.go internal/mcp/tools_patch.go pkg/k8score/workload.go` → 0 hits (no kind allowlist or pod-security check on the apply/patch path) (verified)
  - *To reach the next level:* Ship an enforced policy on by default.
- **B L0:** A privileged pod with hostPath / and host namespaces is node-root, with access to service-account tokens and Secrets the identity can mount. — [pkg/k8score/workload.go:286-321](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/pkg/k8score/workload.go#L286-L321); [internal/mcp/tools_apply.go:56](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools_apply.go#L56) (verified)
  - *To reach the next level:* Restrict what created workloads can reach (namespace, no host access, no Secret mounts).
- **Cap:** none
- **Notes:** diagnose(in_cluster=true) creates Radar-defined probe Jobs from a first-party image (internal/reachability/runner.go:35); that path is fixed, not model-defined. C4-HOSTROOT not applied: there is no default sandbox; the model chooses privilege.

### C5 Untrusted input blast radius — 0.25 (high)

Radar returns cluster data (pod logs, events, annotations, CRD status, ConfigMaps) that anyone running a workload can influence. Results are structured JSON, Secret values are stripped and logs and values are scrubbed for common token patterns, but nothing marks content as untrusted for the host. A read-only endpoint exists that drops the state-change leg, yet the default endpoint gives the same session untrusted input, write tools and cluster secrets. A hijacked agent can therefore both make irreversible changes and exfiltrate data, for example by applying a pod that mounts Secrets and sends them out.

- **S L2:** Outputs are JSON documents separating fields from content, with Secret data dropped and regex redaction, but no provenance/untrusted flag on log or annotation text. — [internal/mcp/tools.go:3359-3369](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools.go#L3359-L3369); [pkg/ai/context/detail.go:16](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/pkg/ai/context/detail.go#L16); searched `rg -n -i 'untrusted'` in `internal/mcp` → 0 hits (no untrusted marker in MCP outputs) (verified)
  - *To reach the next level:* Mark attacker-influenceable fields (logs, annotations, events, status messages) with an untrusted flag the host can act on.
- **C L2:** The same JSON serialization and redaction apply to every tool's output, but no source is distinguished as untrusted. — [internal/mcp/tools.go:3359-3369](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools.go#L3359-L3369); [pkg/ai/context/logs.go:222](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/pkg/ai/context/logs.go#L222) (verified)
  - *To reach the next level:* Tag every untrusted source, including events and CRD status.
- **D L2:** Structured output is always on; the Rule-of-Two reduction (/mcp-readonly) is available but not the default endpoint. — [internal/app/bootstrap.go:566-567](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/app/bootstrap.go#L566-L567); [docs/mcp.md:81](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/docs/mcp.md#L81) (verified)
  - *To reach the next level:* Make the read-only (no state change) mode the default for agents.
- **B L0:** In the default /mcp session a hijacked agent can apply arbitrary workloads (irreversible change) and run pods that read Secrets and reach the network (exfiltration) with no server-side human step. — [pkg/k8score/workload.go:286-321](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/pkg/k8score/workload.go#L286-L321); [internal/mcp/tools.go:542-544](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools.go#L542-L544); searched `rg -n -i 'privileged|hostPath|hostNetwork|PodSecurity|allowedKinds|kindAllow|denyKind'` in `internal/mcp/tools_apply.go internal/mcp/tools_patch.go pkg/k8score/workload.go` → 0 hits (no kind allowlist or pod-security check on the apply/patch path) (verified)
  - *To reach the next level:* Break the trifecta by default, e.g. no write tools in sessions that read untrusted cluster content.
- **Cap:** C5-WORSTCASE — Worst case in the default configuration is unattended leak plus irreversible action (B L0).
- **Notes:** The in-app investigation runner uses a read-only MCP mount and a system prompt stating cluster data is untrusted (pkg/investigation/prompt.go:134); prompt text is not credited.

### C6 Memory, context & configuration integrity — 0.55 (high)

The MCP server has no memory tool, and Radar's security settings come only from user scope (~/.radar/config.json and your kubeconfig), never from a workspace. The one model-influenced persistence is the in-app investigation history: transcripts (including untrusted tool results) are kept in a 0600 SQLite file for up to 30 days and replayed into follow-up turns of the same investigation. It is per run, visible in the UI and clearable, and a write follow-up runs in a fresh session bound to user-confirmed fix text, but stored content is not validated.

- **S L2:** Persisted run transcripts carry server-authored evidence references; config is read only from the user's home directory, but stored transcript content is not validated before replay. — [internal/config/config.go:139](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/config/config.go#L139); [internal/ai/store.go:106](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/ai/store.go#L106) (verified)
  - *To reach the next level:* Gate or validate what is persisted and replayed (e.g. human review of the saved story) with integrity protection.
- **C L2:** The run store is the only model-influenced persistence and it is per-run; there are no auto-loaded workspace files. — [internal/ai/runs.go:569-575](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/ai/runs.go#L569-L575) (verified)
  - *To reach the next level:* Provenance-tag replayed transcript content end to end.
- **D L2:** History is keyed per run on a single-user local install with 30-day retention by default. — [internal/ai/runs.go:179](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/ai/runs.go#L179) (verified)
  - *To reach the next level:* Enforce that the model cannot influence other runs' namespaces and add shorter default retention.
- **B L3:** Poisoned history only affects later turns of the same run; it is inspectable and purgeable, and write turns need user-confirmed fix text. — [internal/ai/store.go:341](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/ai/store.go#L341); [internal/server/ai_diagnose.go:452-455](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/server/ai_diagnose.go#L452-L455) (verified)
  - *To reach the next level:* Persist only after human review with rollback.
- **Cap:** none

### C7 Third-party extensions — 0.35 (high)

The MCP server loads no plugins or remote code. Radar does, however, launch third-party agent CLIs (Claude Code, Codex, Cursor Agent, OpenCode) that it finds on your PATH, or any binary named by RADAR_AI_CLI_BIN, to run in-app investigations after a consent prompt. Nothing pins or verifies those binaries. In the default safeguarded profile for Claude and Codex the CLI gets a scrubbed environment and only Radar's tools; Cursor and OpenCode are supported only in a full-local profile that inherits your environment and auto-approves.

- **S L1:** Agent CLIs are resolved by fixed name through PATH (or an env override) with no version pin or hash check. — [internal/ai/detect.go:34](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/ai/detect.go#L34); [internal/ai/diagnoser.go:245](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/ai/diagnoser.go#L245) (verified)
  - *To reach the next level:* Pin or verify the agent binary (version/hash allowlist).
- **C L1:** Only one extension type exists (agent CLIs); in full-local profile the CLI also loads the user's own MCP servers unverified. — [internal/ai/detect.go:79](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/ai/detect.go#L79) (verified)
  - *To reach the next level:* Verify every launched component, including MCP servers the CLI loads.
- **D L2:** Investigations run only after a versioned per-profile consent stored in user scope, but the consent does not show the exact binary or version that will run. — [internal/server/ai_diagnose.go:163-173](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/server/ai_diagnose.go#L163-L173) (verified)
  - *To reach the next level:* Show the exact executable path/version and permissions at consent time.
- **B L2:** Safeguarded profile: separate process, scrubbed env, built-in tools disabled, MCP restricted to Radar; full-local (only option for Cursor/OpenCode) inherits full env with --auto. — [internal/ai/agent_claude.go:44-58](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/ai/agent_claude.go#L44-L58); [internal/ai/agent_claude.go:75-77](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/ai/agent_claude.go#L75-L77); [internal/ai/agent_opencode.go:32](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/ai/agent_opencode.go#L32) (verified)
  - *To reach the next level:* Sandbox the CLI per run with its own scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.50 (high)

Radar is careful about what reaches the model: Secret values are never returned, environment variables, Helm values, CRD fields and logs are scrubbed for common token patterns, and usage statistics are opt-in counts only. Radar holds no API keys of its own by default; it uses your kubeconfig without exposing it. Two gaps remain: every MCP call's full arguments are printed to the terminal log, so a Secret applied through apply_resource is logged in plaintext, and cluster Secrets remain reachable indirectly (a model can apply a pod that prints them, defeated only by pattern-based log redaction).

- **S L2:** Structural Secret stripping plus regex redaction on model-bound paths; optional integration tokens stored plaintext 0600 in ~/.radar/config.json. — [pkg/ai/context/detail.go:16](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/pkg/ai/context/detail.go#L16); [pkg/ai/context/redact.go:12](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/pkg/ai/context/redact.go#L12); [internal/config/config.go:62-63](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/config/config.go#L62-L63) (verified)
  - *To reach the next level:* Redact before logs as well as model-bound output, and keep stored tokens in a keychain/secret manager.
- **C L2:** Model-bound outputs and telemetry are covered and agent subprocess env is scrubbed (safeguarded), but the tool-call log prints raw arguments. — [internal/mcp/agentlog.go:125-126](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/agentlog.go#L125-L126); [internal/mcp/toolparams.go:307](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/toolparams.go#L307) (verified)
  - *To reach the next level:* Redact tool arguments in the logging path too.
- **D L2:** Usage reporting is opt-in and content-free, but full-argument logging of every tool call is on by default. — [internal/usagedata/usagedata.go:1](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/usagedata/usagedata.go#L1); [internal/mcp/agentlog.go:125-126](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/agentlog.go#L125-L126) (verified)
  - *To reach the next level:* Make argument logging redacted or opt-in so redaction is always on.
- **B L2:** No Radar-held keys enter context, but long-lived cluster Secrets are reachable through write tools and pod logs. — [pkg/k8score/workload.go:286-321](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/pkg/k8score/workload.go#L286-L321); [pkg/ai/context/logs.go:222](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/pkg/ai/context/logs.go#L222) (verified)
  - *To reach the next level:* Keep reachable credentials scoped and short-lived (e.g. block Secret-mounting workloads from MCP).
- **Cap:** none

### C9 Audit & traceability — 0.45 (high)

Every MCP tool is wrapped by one logging function that prints the tool name, the full arguments, success or error, and duration, plus a structured logfmt line. That gives a usable per-call trace, but it goes only to the process's standard output, records no caller identity, and is lost when the terminal closes. Kubernetes' own audit log and Radar's field manager on writes are the durable record, and those live outside Radar.

- **S L2:** Per-call record of tool, arguments, status and timestamp (log default flags) via logToolCall. — [internal/mcp/agentlog.go:120-153](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/agentlog.go#L120-L153); searched `rg -n -i 'actor|principal|user='` in `internal/mcp/agentlog.go` → 0 hits (no identity in the log lines) (verified)
  - *To reach the next level:* Add actor attribution (requesting user/agent, session) and correlation IDs.
- **C L2:** All 33 tool registrations pass through logToolCall; resource reads (cluster://) are not logged and there are no approvals to record. — searched `rg -n 'logToolCall\("'` in `internal/mcp/tools.go` → 33 hits (every tool registration) (verified)
  - *To reach the next level:* Record resource reads and configuration changes as well.
- **D L2:** On by default and written by the server process the model cannot control, but only to stdout. — [internal/mcp/agentlog.go:125-126](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/agentlog.go#L125-L126) (verified)
  - *To reach the next level:* Write to a durable store outside the agent's reach that cannot be disabled silently.
- **B L1:** log.Printf to the terminal is best-effort and not persisted; records vanish when the process exits. — [internal/mcp/agentlog.go:48-52](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/agentlog.go#L48-L52) (verified)
  - *To reach the next level:* Persist records per action durably.
- **Cap:** none

### C10 Limits & kill switch — 0.33 (high)

Several tools enforce server-side caps (events up to 100, changes up to 50, top up to 100, multi-pod logs 32 KB, bounded diagnose bundles, rightsizing scans up to 3 minutes). Others are open-ended: list_resources has no limit, single-pod log tail length and drain timeout are whatever the caller asks, and there is no rate or concurrency limit on tool calls, including writes. Radar's own investigation runner adds a 15-minute turn timeout, 15 model turns and 3 concurrent runs, but that does not bound an external agent using /mcp.

- **S L2:** Server-enforced caps on some operations (limits clamped, log bundle byte cap). — [internal/mcp/tools.go:1451-1452](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools.go#L1451-L1452); [internal/mcp/response_guards.go:16](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/response_guards.go#L16) (verified)
  - *To reach the next level:* Cap every operation and add rate/concurrency limits.
- **C L1:** Caps cover only a subset of tools; list_resources and get_pod_logs tail_lines are unbounded. — [internal/mcp/tools.go:1999-2002](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools.go#L1999-L2002); [internal/mcp/tools.go:839](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools.go#L839) (verified)
  - *To reach the next level:* Extend caps to every tool.
- **D L1:** Defaults exist (tail 200, drain 60s) but the caller can raise them without ceiling. — [internal/mcp/tools_workloads.go:721-727](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools_workloads.go#L721-L727); [internal/mcp/tools_workloads.go:770-771](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/mcp/tools_workloads.go#L770-L771) (verified)
  - *To reach the next level:* Add hard ceilings the caller cannot exceed.
- **B L1:** No ceiling on how many write calls a runaway agent can issue; the in-app runner's 15-minute turns are generous. — searched `rg -n -i 'rate\.NewLimiter|ratelimit'` in `internal/mcp` → 0 hits (no rate limiting); [internal/ai/runs.go:184](https://github.com/skyhook-io/radar/blob/09719d88ba15c66b5b772b4c483b028c3c2f55ab/internal/ai/runs.go#L184) (verified)
  - *To reach the next level:* Rate-limit mutations and cancel in-flight work on stop.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Pod logs, events, annotations, CRD status returned as tool output (internal/mcp/tools.go:3359) · [B] sensitive data/systems: Operator kubeconfig identity over all namespaces (internal/k8s/context_client.go:57) · [C] state change / egress: apply_resource/patch_resource/manage_* on default /mcp (internal/mcp/tools.go:593) · Same default session? Yes

## Highest-impact improvements
1. Make /mcp read-only by default (serve write tools only behind an explicit --mcp-allow-writes flag) and point documentation at it. — C3 D L2→L3, +0.050 before caps (Playbook 3)
2. Enforce a restricted pod-security profile and kind allowlist on apply_resource/patch_resource. — C4 S L0→L2, +0.150 before caps (Playbook 3 step 1)
3. Add dry-run previews to manage_workload, manage_node, manage_cronjob and manage_rollout. — C2 S L2→L3, +0.075 before caps (Playbook 5)
4. Redact tool arguments before logging them (at least apply_resource YAML and patch bodies). — C8 D L2→L3, +0.050 before caps (Playbook 4)
5. Persist an MCP audit log with caller identity outside the process's stdout. — C9 B L1→L2, +0.050 before caps (Playbook 1 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the MCP server in the default local binary mode; the in-cluster Helm deployment (no-auth and auth modes) was reviewed only for C1's alternative mechanism.
- The in-app AI investigation runner (internal/ai) was reviewed for C6, C7 and C10 context only; the web UI's exec terminal, port-forward, Helm write and local-terminal features are outside the MCP trust boundary and were not scored.
- Cluster-side admission controls (Pod Security Admission, policy engines) can mitigate C4/C5 but are not part of Radar and were not credited.
- No reviewer-injection text was found in the repository.
