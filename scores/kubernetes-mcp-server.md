# Defense-in-Depth Score: Kubernetes MCP Server (containers)

**Repo:** https://github.com/containers/kubernetes-mcp-server · **Commit:** `26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40` · **Reviewed:** 2026-10-03
**What it is:** Native Go MCP server for Kubernetes and OpenShift (pods, resources, Helm)
**Category:** Infrastructure & Ops
**Scored configuration:** Local stdio server launched as the README leads (npx -y kubernetes-mcp-server@latest) with no config file: default toolsets core+config, read_only=false, no confirmation rules, credentials from the operator's kubeconfig.
**Agent surface (default):** code execution yes · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 4.4 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L3 | L3 | L0 | L2 | 0.55 | G1 | **0.50** (alt) | High |
| C2 | Approval gates | L2 | L3 | L3 | L0 | 0.53 | — | **0.53** | High |
| C3 | Tool & action scoping | L1 | L2 | L2 | L0 | 0.33 | — | **0.33** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L0 | 0.07 | C5-WORSTCASE | **0.07** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L2 | L2 | L0 | 0.33 | — | **0.33** | High |
| C9 | Audit & traceability | L2 | L2 | L0 | L1 | 0.35 | G1 | **0.35** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | Medium |

Controls where a risk surface exists: 2.35 / 8.0 (29%); 2 criteria scored SA (surface absent).

As shipped, this server hands the model the operator's full kubeconfig authority across every context in it: it can exec arbitrary commands in any pod, run any image, apply or delete any resource, and read Secrets and the raw kubeconfig (including tokens and client keys) through configuration_view. Every tool carries accurate read-only/destructive annotations and there are solid opt-in controls (read_only mode, elicitation-based confirmation rules, a central denied-resources filter, OAuth with token exchange), but none of them is on by default. The dominant risk is a prompt-injected or misled host agent using cluster-admin-equivalent credentials unattended, with nothing in the server's default configuration to stop or record it.

## Critical gaps
- Default local mode gives the model the operator's full kubeconfig authority across every context, with no narrowing. (ASI03, T3; C1) — [pkg/kubernetes/provider.go:151-165](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/provider.go#L151-L165); [pkg/kubernetes/provider_kubeconfig.go:16](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/provider_kubeconfig.go#L16)
- MCP token passthrough is the default cluster auth mode in HTTP deployments: the client's bearer token is forwarded to the Kubernetes API. (ASI03, T9; C1) — [pkg/config/config.go:503-507](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L503-L507); [pkg/kubernetes/manager.go:206-216](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/manager.go#L206-L216)
- pods_exec runs model-chosen commands in any existing workload container, and arbitrary pods can be created, with no isolation provided by the server. (ASI05, T11; C4) — [pkg/kubernetes/pods.go:266-283](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/pods.go#L266-L283); [pkg/toolsets/core/pods.go:204-205](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/toolsets/core/pods.go#L204-L205)
- Worst case under injection: the default tool set can read credentials (configuration_view, Secrets) and both exfiltrate and delete cluster state with no server-side human step. (ASI01, LLM01; C5) — [pkg/kubernetes/configuration.go:66-82](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/configuration.go#L66-L82); [pkg/config/config.go:411](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L411)

## Criterion details

### C1 Identity & least privilege — 0.50 (high)

In the default local mode the server uses whatever credentials are in the operator's kubeconfig, and the model can pick any context in that file per call, so it inherits the operator's full authority across every cluster they can reach. In HTTP mode the default auth mode forwards the client's bearer token straight to the Kubernetes API (token passthrough), and HTTP-mode access control is not otherwise locked down. An opt-in OAuth mode with token exchange gives each request the requesting user's own cluster identity and fails closed when no token is present, which is a genuinely strong design, but it is off by default and only applies to HTTP deployments.

- **default configuration** (default; raw 0.07, cap C1-PASSTHRU → 0.07)
  - **S L0:** Default credentials are the operator's ambient kubeconfig (kubeconfig provider auto-selected), with no narrowing of verbs or resources. — [pkg/kubernetes/provider.go:151-165](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/provider.go#L151-L165); [pkg/kubernetes/manager.go:193-198](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/manager.go#L193-L198) (verified)
    - *To reach the next level:* No dedicated identity: the server never obtains its own scoped credential in the default local mode.
  - **C L1:** Every Kubernetes API request passes one central round tripper, but by default it authorizes everything (empty denied list) and the model chooses which kubeconfig context to use on each call. — [pkg/kubernetes/accesscontrol_round_tripper.go:275-278](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/accesscontrol_round_tripper.go#L275-L278); [pkg/mcp/tools_gosdk.go:98](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/mcp/tools_gosdk.go#L98); [pkg/kubernetes/provider_kubeconfig.go:16](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/provider_kubeconfig.go#L16) (verified)
    - *To reach the next level:* No authorization layer narrows what built-in tools can do with the credential; per-call context selection is unrestricted.
  - **D L0:** Default install runs with whatever the kubeconfig grants (commonly cluster-admin) and read_only defaults to false. — [pkg/config/config.go:411](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L411); [pkg/config/config.go:503-507](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L503-L507) (verified)
    - *To reach the next level:* No least-privilege default; narrowing requires the operator to write a config file or supply a scoped kubeconfig.
  - **B L0:** A hijacked session holds the operator's full authority over every cluster in the kubeconfig, including writes, deletes, and pod exec. — [pkg/kubernetes/provider_kubeconfig.go:16](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/provider_kubeconfig.go#L16); [pkg/config/config.go:416](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L416) (verified)
    - *To reach the next level:* Blast radius is not limited to one cluster, one namespace, or read-mostly access.
- **opt-in require_oauth with token_exchange (HTTP mode)** (alt; raw 0.55, cap G1 → 0.50) ← counted
  - **S L3:** With OAuth required and token exchange configured, each request's user token is exchanged (RFC 8693 / Entra OBO / Keycloak) for a cluster token on behalf of that user before any client is built. — [pkg/kubernetes/provider_token_exchange.go:35-55](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/provider_token_exchange.go#L35-L55); [pkg/kubernetes/manager.go:206-216](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/manager.go#L206-L216) (verified)
    - *To reach the next level:* Exchanged authority is not intersected with a server-side least-privilege policy by default and is not revoked after use.
  - **C L3:** Every tool obtains its client through GetDerivedKubernetes, and require_oauth makes tokenless requests fail closed. — [pkg/mcp/handler_params.go:10](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/mcp/handler_params.go#L10); [pkg/kubernetes/manager.go:193-196](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/manager.go#L193-L196); [pkg/kubernetes/provider_token_exchange.go:77-81](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/provider_token_exchange.go#L77-L81) (verified)
    - *To reach the next level:* Not every policy-resolution failure is shown to deny: a missing token endpoint logs a warning and skips exchange.
  - **D L0:** require_oauth defaults to false and token exchange is unset by default. — [pkg/config/config.go:433](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L433) (verified)
    - *To reach the next level:* Not on in the scored configuration.
  - **B L2:** A hijacked session is bounded by the requesting user's own RBAC on the target cluster, which typically includes writes. — [pkg/kubernetes/manager.go:206-222](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/manager.go#L206-L222) (verified)
    - *To reach the next level:* Not limited to read-mostly or non-destructive writes.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.
- **Notes:** resources_create_or_update can write RoleBindings/ClusterRoleBindings, but Kubernetes' own RBAC escalation prevention stops a credential granting more than it already holds, so C1-SELFESC was not applied. The Helm chart creates a ServiceAccount with no RBAC bindings by default (charts/kubernetes-mcp-server/values.yaml), a sound in-cluster default that is not the README's lead mode.

### C2 Approval gates — 0.53 (high)

The server owns no approval loop itself, so it is rated on what it gives the host. Read and write operations are separate tools and every tool in every toolset carries read-only and destructive annotations, with pod exec, deletes, scale, and create-or-update all flagged as non-read-only. The server also offers a server-enforced read-only mode and elicitation-based confirmation rules, but both are off by default, there is no dry-run or preview for destructive operations, and when confirmation rules are configured a client without elicitation support is allowed through by default. A wrongly approved delete or full-replace apply is irreversible.

- **S L2:** Separate read and write tools with readOnlyHint/destructiveHint on every tool; pods_exec is flagged destructive. — [pkg/toolsets/core/pods.go:231-235](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/toolsets/core/pods.go#L231-L235); [pkg/toolsets/core/resources.go:130-134](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/toolsets/core/resources.go#L130-L134); searched `rg -n -i 'dry.?run' -g '!*_test.go'` in `pkg` → 1 hits (the only hit sets Helm install.DryRun = false; no dry-run is offered to callers) (verified)
  - *To reach the next level:* No preview or dry-run for destructive operations (resources_create_or_update, resources_delete, helm).
- **C L3:** Every mutating tool in the default toolsets lacks readOnlyHint, and read_only mode excludes any tool not explicitly annotated read-only, so the annotation signal covers every path. — [pkg/mcp/mcp.go:70-75](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/mcp/mcp.go#L70-L75); [pkg/toolsets/core/pods.go:332-335](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/toolsets/core/pods.go#L332-L335) (verified)
  - *To reach the next level:* pods_run is marked non-destructive although it server-side-applies a Pod and Service that can update existing objects of the same name; no parsed per-argument policy.
- **D L3:** Annotations are hard-coded per tool and tool_overrides can only change descriptions, so neither the model nor a workspace file can alter the risk signal. — [pkg/config/config.go:39-42](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L39-L42); [pkg/config/config.go:421-424](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L421-L424); [pkg/confirmation/confirmation.go:56-62](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/confirmation/confirmation.go#L56-L62) (verified)
  - *To reach the next level:* The server-enforced read-only mode and confirmation rules are opt-in, and confirmation_fallback defaults to allow.
- **B L0:** Default tools can delete arbitrary resources and replace resources via server-side apply in any cluster in the kubeconfig, with no undo. — [pkg/toolsets/core/resources.go:141](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/toolsets/core/resources.go#L141); [pkg/toolsets/core/resources.go:118-119](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/toolsets/core/resources.go#L118-L119) (verified)
  - *To reach the next level:* No rollback, preview, or quantity bounds on consequential actions.
- **Cap:** none
- **Notes:** Opt-in confirmation_rules (tool-level and Kubernetes verb/kind-level) run inside the server via MCP elicitation (pkg/mcp/tools_gosdk.go:90-95, pkg/kubernetes/confirmation_validator.go). Without a dry-run they would not raise S above L2, so they were not scored as a separate alt.

### C3 Tool & action scoping — 0.33 (high)

The default tools are general-purpose: pods_exec runs any command array in any pod, pods_run starts any image, and resources_create_or_update applies any manifest of any kind. Inputs are typed JSON schemas, and a central round tripper can enforce a denied-resources list on every Kubernetes API call, but that list is empty by default, is a denylist of resource kinds only, and pod exec can still read data (such as mounted Secrets) whose kind is denied. Toolsets are selectable, but the default set includes write and exec tools.

- **S L1:** The only argument-level control is a GVK denylist checked in the HTTP round tripper; commands, images, and manifests are otherwise passed through. — [pkg/kubernetes/accesscontrol_round_tripper.go:177-178](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/accesscontrol_round_tripper.go#L177-L178); [pkg/kubernetes/pods.go:278-281](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/pods.go#L278-L281) (verified)
  - *To reach the next level:* No allowlists on namespaces, kinds, images, or commands, and no numeric bounds.
- **C L2:** The denylist runs for every client built from the wrapped rest.Config, but pod exec reaches data inside containers that the kind-level list cannot see. — [pkg/kubernetes/kubernetes.go:93-106](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/kubernetes.go#L93-L106); [pkg/toolsets/core/pods.go:204-205](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/toolsets/core/pods.go#L204-L205) (verified)
  - *To reach the next level:* Coverage is limited by the mechanism: exec into pods bypasses kind-level restrictions on Secrets and similar data.
- **D L2:** Toolsets are selectable, but the default core toolset includes exec, run, apply, and delete tools. — [pkg/config/config.go:416](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L416); [pkg/config/config.go:410](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L410) (verified)
  - *To reach the next level:* Default tool set is not read-only.
- **B L0:** Misused tools act on any resource, namespace, image, or command in production clusters. — [pkg/toolsets/core/pods.go:308-309](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/toolsets/core/pods.go#L308-L309) (verified)
  - *To reach the next level:* No workspace-style scoping or quantity bounds.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

The server never runs code on its own host, but pods_exec executes model-chosen commands inside whatever existing container the model names, and pods_run or resources_create_or_update can start any image with any pod spec. There is no sandbox the model cannot redefine: the execution target is a production workload carrying its own service-account token, network access, and mounts, and the model can also create a privileged pod with host mounts if the credentials allow it. Nothing in the server constrains this.

- **S L0:** Commands run directly in the chosen workload container via the exec subresource; no isolation primitive is provided by the server. — [pkg/kubernetes/pods.go:266-283](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/pods.go#L266-L283) (verified)
  - *To reach the next level:* No separate execution environment the model cannot choose or redefine (e.g., a dedicated hardened debug pod).
- **C L0:** Neither pods_exec, pods_run, nor arbitrary workload creation passes through any isolation layer. — [pkg/kubernetes/pods.go:313](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/pods.go#L313) (verified)
  - *To reach the next level:* The main exec path is not sandboxed.
- **D L0:** No sandbox exists to enable; exec tools are in the default toolset. — [pkg/config/config.go:416](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L416) (verified)
  - *To reach the next level:* No isolation control exists at any setting.
- **B L0:** Executed commands reach the target workload's credentials and network, and the model can schedule privileged or host-mounting pods with the operator's credentials. — [pkg/toolsets/core/resources.go:118-126](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/toolsets/core/resources.go#L118-L126) (verified)
  - *To reach the next level:* Execution is host-equivalent in the cluster when the kubeconfig allows privileged pods.
- **Cap:** none

### C5 Untrusted input blast radius — 0.07 (high)

Pod logs, exec output, events, and resource contents (annotations, ConfigMaps) are returned as plain text with no provenance or untrusted marker, so anything a workload writes reaches the model on equal footing with real data. The server offers a read-only mode that would drop the state-change leg, but it is opt-in. In the default configuration a hijacked host agent can read Secrets and the raw kubeconfig and exfiltrate them (for example by running a pod that posts them out) and delete or replace resources, all without the server asking anyone.

- **S L1:** Outputs are plain text (pod logs, exec stdout) with no source tagging or untrusted flag. — [pkg/toolsets/core/pods.go:483-499](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/toolsets/core/pods.go#L483-L499); searched `rg -n -i 'untrusted|provenance' -g '!*_test.go'` in `pkg` → 0 hits (verified)
  - *To reach the next level:* No structured separation of untrusted content from metadata on the log/exec paths.
- **C L0:** No untrusted source is distinguished from any other output. — searched `rg -n -i 'untrusted|provenance' -g '!*_test.go'` in `pkg` → 0 hits (verified)
  - *To reach the next level:* No source (logs, events, resource fields) carries an untrusted marker.
- **D L0:** The read-only mode that would remove the state-change leg is off by default. — [pkg/config/config.go:411](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L411) (verified)
  - *To reach the next level:* No Rule-of-Two leg is dropped by default.
- **B L0:** A hijacked session can read credentials via configuration_view or Secret reads and both exfiltrate and destroy cluster state unattended. — [pkg/kubernetes/configuration.go:66-82](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/configuration.go#L66-L82); [pkg/toolsets/core/resources.go:141](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/toolsets/core/resources.go#L141) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions are both available without any server-side human step.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 1.00 (high)

The server keeps no memory, conversation store, or retrieval index, and it does not load instruction or configuration files from the working directory. Configuration comes only from an explicit --config/--config-dir path or the MCP_CONFIG_PATH environment variable, and credentials from the user's kubeconfig, all user scope. Cluster state the model writes and later reads back is untrusted input, covered in C5.

- **Structural absence:** searched `rg -n 'godotenv|os\.Getwd|os\.WriteFile' -g '!*_test.go'` in `pkg cmd` → 0 hits (no dotenv loading, no working-directory lookup, no file writes by the server); [pkg/kubernetes-mcp-server/cmd/root.go:115-116](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes-mcp-server/cmd/root.go#L115-L116)

### C7 Third-party extensions — 1.00 (high)

The server loads no plugins and launches no MCP servers or local processes; it is a single Go binary talking to the Kubernetes API. Container images the model asks pods_run to start run inside the cluster, not in the server, and are scored as code execution under C4 and tool scope under C3.

- **Structural absence:** searched `rg -n 'plugin\.Open|"os/exec"|exec\.Command' -g '!*_test.go'` in `pkg cmd` → 0 hits (no plugin loading or subprocess launch in server code)

### C8 Secrets & sensitive-data protection — 0.33 (high)

Protocol logs pass through a regex-based sanitizer and sensitive config options are redacted when the configuration is dumped, and telemetry is off unless an endpoint is set. But the model-bound path is unprotected: the default configuration_view tool returns the flattened kubeconfig without redaction (tokens and client keys included) in stdio mode, and Secret objects come back in full through the generic resource tools. Those are long-lived, often cluster-admin credentials.

- **S L1:** Log sanitization is a regex denylist; nothing masks credentials in tool results sent to the model, and configuration_view deliberately returns them. — [pkg/mcplog/log.go:125](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/mcplog/log.go#L125); [pkg/kubernetes/configuration.go:66-82](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/configuration.go#L66-L82); searched `rg -n -i 'shortenconfig|DATA\+OMITTED|REDACTED'` in `pkg/kubernetes pkg/toolsets/config pkg/toolsets/core` → 0 hits (verified)
  - *To reach the next level:* No redaction before model-bound messages (e.g., ShortenConfig on configuration_view, Secret data masking).
- **C L2:** Protocol logs and config dumps are sanitized; configuration_view is hidden only in HTTP mode. — [pkg/mcp/middleware.go:66-68](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/mcp/middleware.go#L66-L68); [pkg/config/option.go:44-47](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/option.go#L44-L47); [pkg/mcp/tool_filter.go:56-61](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/mcp/tool_filter.go#L56-L61) (verified)
  - *To reach the next level:* Model-bound tool results are not covered in stdio mode.
- **D L2:** Telemetry is off unless an OTLP endpoint is configured, log level defaults to 0, and sanitization is always applied to logs. — [pkg/config/config.go:131-136](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L131-L136); [pkg/config/config.go:395](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L395) (verified)
  - *To reach the next level:* Redaction is best-effort regex and absent from the model path, so it does not meet always-on redaction.
- **B L0:** The model can obtain long-lived, high-privilege kubeconfig credentials for every context on request. — [pkg/toolsets/config/configuration.go:176-182](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/toolsets/config/configuration.go#L176-L182) (verified)
  - *To reach the next level:* Credentials reachable by the model are neither scoped nor short-lived.
- **Cap:** none
- **Notes:** C8-MODELSECRETS was not applied: credentials enter model context only when configuration_view or a Secret read is called, not routinely in every prompt. The opt-in KubeVirt troubleshoot tool redacts cloud-init secrets before returning them.

### C9 Audit & traceability — 0.35 (high)

In the default configuration the server keeps no record of tool calls: names and arguments are logged only at verbosity 5 and results at 6, while the default verbosity is 0, and OpenTelemetry tracing is off unless an endpoint is configured. When enabled, logs carry tool name, sanitized arguments, and results, and spans carry tool name and status, but there is no actor attribution beyond user agent and no tamper-evidence. Kubernetes API audit logs on the cluster side are not the server's own record.

- **S L2:** When verbosity is raised, each tool call is logged with name and sanitized arguments, and results at V(6); OTel spans record tool name and status. — [pkg/mcp/middleware.go:69-91](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/mcp/middleware.go#L69-L91) (verified)
  - *To reach the next level:* No requesting-principal or approver attribution in the record.
- **C L2:** The logging middleware wraps every tools/call, so all built-in tools are covered when enabled. — [pkg/mcp/mcp.go:184](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/mcp/mcp.go#L184) (verified)
  - *To reach the next level:* Confirmation approvals and denials are not recorded as distinct events.
- **D L0:** Default log_level is 0 and telemetry requires an endpoint, so tool calls are not recorded by default. — [pkg/config/config.go:395](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L395); [pkg/config/config.go:131-136](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L131-L136) (verified)
  - *To reach the next level:* Recording is opt-in.
- **B L1:** Logging is best-effort klog output; actions proceed regardless of whether anything is written. — [pkg/mcp/middleware.go:91-93](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/mcp/middleware.go#L91-L93) (verified)
  - *To reach the next level:* No per-action durable record.
- **Cap:** G1 — Tool-call recording exists only above the default log level or with telemetry configured.

### C10 Limits & kill switch — 0.25 (medium)

The server bounds little of its own work. Pod logs default to the last 100 lines but the caller can raise that, list and get operations have no size limits, exec output is buffered without a cap or timeout, and the per-session rate limiter is disabled by default. Request contexts are passed to exec streaming, so a cancelled MCP request should stop in-flight exec, but nothing enforces a ceiling.

- **S L1:** Limits are caller-chosen (log tail) and the rate limiter is opt-in; no output-size caps or tool timeouts. — [pkg/kubernetes/pods.go:136-142](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/pods.go#L136-L142); [pkg/mcp/middleware.go:374-377](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/mcp/middleware.go#L374-L377); searched `rg -n '"limit"'` in `pkg/toolsets/core` → 0 hits (verified)
  - *To reach the next level:* No server-enforced output caps or timeouts on list, get, log, or exec operations.
- **C L1:** Only pod logs have a default bound; other tools have none. — [pkg/kubernetes/pods.go:28](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/pods.go#L28) (verified)
  - *To reach the next level:* No per-tool timeout or cap across the tool set.
- **D L1:** The rate limit defaults to 0 (disabled) and the log tail default can be raised by the model. — [pkg/config/config.go:68](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/config/config.go#L68) (verified)
  - *To reach the next level:* No sensible enforced default the model cannot raise.
- **B L1:** Exec streams honour the request context, so cancellation from the go-sdk (assumed to cancel the handler context on notifications/cancelled) stops in-flight exec, but there is no ceiling on what runs before that. — [pkg/kubernetes/pods.go:313-315](https://github.com/containers/kubernetes-mcp-server/blob/26eaf54c2a67fd4ea42657a83e6fd22aca4ccb40/pkg/kubernetes/pods.go#L313-L315) (inferred)
  - *To reach the next level:* No time or volume ceilings on server work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Pod logs, exec output, events, and resource fields returned verbatim (pkg/toolsets/core/pods.go:483-499) · [B] sensitive data/systems: Raw kubeconfig credentials and Secret objects (pkg/kubernetes/configuration.go:66-82) · [C] state change / egress: pods_exec, pods_run, resources_create_or_update, resources_delete in the default toolset (pkg/toolsets/core/resources.go:118-141) · Same default session? Yes

## Highest-impact improvements
1. Redact credentials in configuration_view (clientcmdapi.ShortenConfig or drop the tool from the default stdio set) and mask Secret data in resource tool output. — C8 S L1→L2, +0.075 before caps
2. Ship read_only=true (or a read-only default toolset) so write and exec tools require explicit operator elevation. — C3 D L2→L3, +0.050 before caps (Playbook 3)
3. Log every tool call (name, target context, arguments, outcome) at the default log level. — C9 D L0→L2, +0.100 before caps (Playbook 1 step 3)
4. Add server-side dry-run previews for resources_create_or_update, resources_delete, and helm operations. — C2 S L2→L3, +0.075 before caps (Playbook 5)
5. Enforce default output-size caps and timeouts on list, log, and exec tools and enable a default rate limit. — C10 S L1→L2, +0.075 before caps

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the README-led local stdio mode; HTTP mode differs (configuration_view hidden) and the Helm chart ships a ServiceAccount with no RBAC bindings.
- Optional toolsets (helm, kiali, kubevirt, tekton, kcp, netobserv) were checked only for annotations; helm_install can deploy charts from model-chosen repositories when enabled.
- Kubeconfig exec credential plugins run by client-go are operator-configured (user scope) and were not treated as extensions.
- C10 B relies on the go-sdk cancelling the handler context on notifications/cancelled (library behaviour, inferred).
- No reviewer-directed prompt injection was found in README, AGENTS.md, SECURITY.md, or .agents skills.
