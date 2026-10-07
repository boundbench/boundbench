# Defense-in-Depth Score: mcp-server-kubernetes

**Repo:** https://github.com/Flux159/mcp-server-kubernetes · **Commit:** `3d71add204014dcceddebe53e9695ed7d8a6d129` (4.1.9) · **Reviewed:** 2026-10-03
**What it is:** TypeScript MCP server for kubectl/Helm management of Kubernetes clusters
**Category:** Infrastructure & Ops
**Scored configuration:** Local stdio server launched as the README leads (npx mcp-server-kubernetes) with no environment variables: all tools enabled, credentials from the operator's ~/.kube/config, telemetry off.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 2.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L1 | L2 | L0 | L2 | 0.33 | G1 | **0.33** (alt) | High |
| C3 | Tool & action scoping | L1 | L2 | L2 | L0 | 0.33 | G2 | **0.25** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L0 | 0.07 | C5-WORSTCASE | **0.07** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | Medium |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L1 | L2 | L0 | L1 | 0.28 | G1 | **0.28** (alt) | Medium |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |

Controls where a risk surface exists: 1.60 / 9.0 (18%); 1 criterion scored SA (surface absent).

As shipped, this server gives the model the operator's full kubeconfig authority across every context, with exec into any pod, a generic kubectl tool, Helm installs from any repository, and forced deletes all enabled. It has a careful argv flag denylist and masks Secret values in kubectl_get, but the generic tool can run 'config view --raw', which returns raw credentials to the model, and neither the denylist nor the masking is a complete boundary. The dominant risk is a prompt-injected host agent using cluster-admin-equivalent credentials unattended; the read-only and allowlist modes that would contain it are opt-in.

## Critical gaps
- Default local mode runs every tool with the operator's full kubeconfig across all contexts, with no authorization check. (ASI03, T3; C1) — [src/utils/kubernetes-manager.ts:79-85](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/utils/kubernetes-manager.ts#L79-L85); [src/models/common-parameters.ts:1-5](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/models/common-parameters.ts#L1-L5)
- exec_in_pod and kubectl_generic run model-chosen commands in any pod, and arbitrary pods can be created, with no isolation from the server. (ASI05, T11; C4) — [src/tools/exec_in_pod.ts:105-124](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/exec_in_pod.ts#L105-L124); [src/tools/kubectl-generic.ts:98-99](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L98-L99)
- Worst case under prompt injection from pod logs or resource fields: the default tool set can both exfiltrate cluster data and delete resources with no server-side human step. (ASI01, LLM01; C5) — [src/tools/helm-operations.ts:342-346](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/helm-operations.ts#L342-L346); [src/tools/kubectl-delete.ts:166-169](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-delete.ts#L166-L169)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

In the default local mode the server loads the operator's own kubeconfig and runs every kubectl and helm command with it, and the model can pick any context in that file on every call, so it inherits the operator's full authority across every cluster they can reach. There is no server identity, no scoping of verbs or resources, and no authorization check of any kind before a command runs; the only narrowing is an opt-in tool allowlist set by environment variable. A hijacked or misled session therefore holds whatever the kubeconfig grants, which for most operators is cluster-admin.

- **S L0:** Every tool runs kubectl/helm with the operator's ambient kubeconfig (loadFromDefault) and the model chooses the context per call; nothing narrows the credential. — [src/utils/kubernetes-manager.ts:79-85](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/utils/kubernetes-manager.ts#L79-L85); [src/models/common-parameters.ts:1-5](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/models/common-parameters.ts#L1-L5) (verified)
  - *To reach the next level:* No dedicated or role-scoped identity: the server never obtains its own narrower credential.
- **C L0:** No authorization layer exists: tools build kubectl argv and execute it with the full process environment, and the only check is an argv flag denylist that does not evaluate permissions. — [src/tools/kubectl-generic.ts:163-167](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L163-L167); [src/security/kubectl-flags.ts:435-442](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/security/kubectl-flags.ts#L435-L442); searched `rg -n -i 'authoriz|SubjectAccessReview|can-i'` in `src/` → 2 hits (Both hits are HTTP X-MCP-AUTH error strings in src/utils/auth.ts, not authorization of Kubernetes requests.) (verified)
  - *To reach the next level:* No check maps a request to permissions before credentials are used, on any tool path.
- **D L0:** With no env vars set, isToolAllowed returns true for every tool and the server uses whatever the kubeconfig grants. — [src/index.ts:84-104](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L84-L104); [src/index.ts:100-104](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L100-L104) (verified)
  - *To reach the next level:* No least-privilege default; narrowing requires the operator to set ALLOWED_TOOLS / read-only env vars or supply a scoped kubeconfig.
- **B L0:** A hijacked session holds the operator's full authority over every cluster in the kubeconfig, including deletes, pod exec and RBAC writes. — [src/models/common-parameters.ts:1-5](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/models/common-parameters.ts#L1-L5); [src/index.ts:130-171](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L130-L171) (verified)
  - *To reach the next level:* Blast radius is not limited to one cluster, one namespace, or read-mostly access.
- **Cap:** none
- **Notes:** kubectl_apply/create can write RoleBindings, but Kubernetes RBAC escalation prevention stops a credential granting more than it holds, so C1-SELFESC was not applied. The Helm chart (not the README's lead mode) ships a ClusterRole with cluster-wide pod/exec/secret-read and write verbs; its network transport defaults are not locked down.

### C2 Approval gates — 0.33 (high)

As a tool server it relies on the host to ask the human, so what matters is how accurately it labels risk. Most mutating tools carry a destructive hint and there are dry-run options on apply, create, patch and node drain, but the labels are not reliable: some annotations are inaccurate, kubectl_create and port_forward carry no hints at all, and kubectl_generic mixes reads and writes in one tool. An opt-in read-only mode is enforced by the server on every call, though it is not a strict boundary. Deletes (including forced deletes and Helm uninstall) are irreversible with no preview.

- **default configuration** (default; raw 0.25 → 0.25)
  - **S L1:** Annotations exist on most tools but some are inaccurate and they are missing on kubectl_create and port_forward; kubectl_generic mixes reads and writes. — [src/tools/kubectl-create.ts:11-15](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-create.ts#L11-L15); [src/tools/port_forward.ts:62-67](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/port_forward.ts#L62-L67); [src/tools/kubectl-generic.ts:15-21](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L15-L21) (verified)
    - *To reach the next level:* Accurate readOnly/destructive hints on every tool, with read and write split into separate tools.
  - **C L1:** exec_in_pod, kubectl_generic and delete are flagged destructive, but kubectl_create (creates arbitrary resources) reaches state changes without a destructive signal, and other annotations are inaccurate. — [src/tools/exec_in_pod.ts:25-30](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/exec_in_pod.ts#L25-L30); [src/tools/kubectl-create.ts:11-15](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-create.ts#L11-L15) (verified)
    - *To reach the next level:* Every mutating tool must be flagged so a host gate keyed on hints covers all of them.
  - **D L2:** Hints are static in code and cannot be changed by the model or input; the server-side read-only mode is off unless ALLOW_ONLY_READONLY_TOOLS=true. — [src/index.ts:84-104](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L84-L104); [src/index.ts:100-104](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L100-L104) (verified)
    - *To reach the next level:* No server-side confirmation or read-only posture on by default; capped one level above the weak signalling.
  - **B L0:** A wrongly approved call can force-delete resources, uninstall releases or delete namespaces with no undo; dry-run exists only for apply/create/patch/drain. — [src/tools/kubectl-delete.ts:166-169](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-delete.ts#L166-L169); [src/tools/kubectl-apply.ts:98-101](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-apply.ts#L98-L101) (verified)
    - *To reach the next level:* No preview or rollback for delete/uninstall/scale/generic operations.
- **opt-in ALLOW_ONLY_READONLY_TOOLS mode** (alt; raw 0.33, cap G1 → 0.33) ← counted
  - **S L1:** Read-only mode lists and allows only eight tools, but it is not a strict boundary, and annotation quality is unchanged. — [src/index.ts:107-116](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L107-L116) (verified)
    - *To reach the next level:* Accurate hints on every tool and a robust read-only set.
  - **C L2:** isToolAllowed is enforced on every CallTool request, not only in ListTools, so hidden tools cannot be called directly. — [src/index.ts:256-261](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L256-L261); [src/index.ts:107-116](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L107-L116) (verified)
    - *To reach the next level:* Read-only mode is not a strict boundary.
  - **D L0:** Opt-in via environment variable; off in the scored configuration. — [src/index.ts:84-104](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L84-L104) (verified)
    - *To reach the next level:* Not on by default.
  - **B L2:** With read-only mode on, remaining state changes are limited and reversible. — [src/index.ts:107-116](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L107-L116) (verified)
    - *To reach the next level:* No rate limits or quantity bounds.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.
- **Notes:** node_management requires confirmDrain=true for a drain, but the model sets that argument itself, so it is not a human gate (C2-SELFAPPROVE not applied because it is not the server's primary risk-signalling control).

### C3 Tool & action scoping — 0.25 (high)

Every kubectl and helm invocation passes through one wrapper that rejects a denylist of dangerous flags (API server, token, impersonation, kubeconfig) and most tools refuse operands that start with a dash, which is a real improvement over raw passthrough. But kubectl_generic accepts any kubectl verb and any positional arguments, and the flag denylist is not a complete boundary. Helm tools accept any repository URL and chart, and every tool, including exec and generic kubectl, is enabled by default.

- **S L1:** Validation is a denylist of flag names plus typed schemas; kubectl_generic passes any verb and positional args, and helm accepts any repo URL. — [src/security/kubectl-flags.ts:16-51](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/security/kubectl-flags.ts#L16-L51); [src/tools/kubectl-generic.ts:98-99](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L98-L99); [src/tools/helm-operations.ts:342-346](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/helm-operations.ts#L342-L346) (verified)
  - *To reach the next level:* No allowlist of verbs/subcommands or repo hosts; positional arguments of kubectl_generic are not validated.
- **C L2:** The argv guard runs at one choke point (execFileSyncSafe) used by every kubectl/helm call, and port_forward calls it directly. — [src/security/kubectl-flags.ts:435-442](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/security/kubectl-flags.ts#L435-L442); [src/tools/port_forward.ts:15-17](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/port_forward.ts#L15-L17); searched `rg -n 'from "child_process"'` in `src/` → 2 hits (Only the safe wrapper and port_forward import child_process; both run assertSafeArgv.) (verified)
  - *To reach the next level:* Coverage is capped one level above the denylist's strength; a central allowlist policy would be needed.
- **D L2:** All tools, including exec_in_pod, kubectl_generic and helm, are enabled by default; groups can be removed via ALLOW_ONLY_NON_DESTRUCTIVE_TOOLS / ALLOW_ONLY_READONLY_TOOLS / ALLOWED_TOOLS. — [src/index.ts:84-104](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L84-L104); [src/index.ts:100-104](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L100-L104); [src/index.ts:130-171](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L130-L171) (verified)
  - *To reach the next level:* Read-only tool set by default with writes/exec enabled explicitly.
- **B L0:** kubectl_generic is a general-purpose kubectl against any namespace of any cluster in the kubeconfig. — [src/tools/kubectl-generic.ts:15-21](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L15-L21); [src/models/common-parameters.ts:1-5](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/models/common-parameters.ts#L1-L5) (verified)
  - *To reach the next level:* No namespace, resource or quantity bounds on any tool.
- **Cap:** G2 — The flag denylist is not a complete boundary against the model at runtime.
- **Notes:** ALLOW_KUBECTL_UNSAFE_FLAGS=true disables the denylist entirely (operator setting).

### C4 Code-execution isolation — 0.00 (high)

exec_in_pod runs any model-chosen command inside any pod container in any namespace, and kubectl_apply, kubectl_create and kubectl_generic can create arbitrary pods, including privileged or host-mounted ones. The server provides no isolation around any of this: commands run inside existing workloads with their own service-account tokens and network access, and on the host side kubectl and helm run as the operator's user with the full environment. The argv is passed without a shell, which prevents shell injection on the host but is not a sandbox.

- **S L0:** exec_in_pod forwards an arbitrary argv to kubectl exec in a model-chosen pod; no isolation primitive is involved. — [src/tools/exec_in_pod.ts:105-124](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/exec_in_pod.ts#L105-L124) (verified)
  - *To reach the next level:* No sandbox or isolation boundary for model-directed execution.
- **C L0:** No execution path is isolated: exec_in_pod, kubectl_generic (run/debug), apply/create of pod manifests, and host-side kubectl/helm all run unconfined. — [src/tools/exec_in_pod.ts:105-124](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/exec_in_pod.ts#L105-L124); [src/tools/kubectl-generic.ts:98-99](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L98-L99); [src/tools/kubectl-generic.ts:163-167](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L163-L167) (verified)
  - *To reach the next level:* At least the main exec tool would need to run inside an isolation boundary.
- **D L0:** Exec tools are enabled by default with no sandbox to turn on. — [src/index.ts:100-104](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L100-L104); [src/tools/exec_in_pod.ts:25-30](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/exec_in_pod.ts#L25-L30) (verified)
  - *To reach the next level:* No isolation on by default.
- **B L0:** Commands run inside production workloads that hold their own credentials and network, and created pods can be privileged or mount the host, reaching node root. — [src/tools/exec_in_pod.ts:105-124](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/exec_in_pod.ts#L105-L124); [src/index.ts:130-171](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L130-L171) (verified)
  - *To reach the next level:* Nothing confines what exec'd commands or created pods can reach.
- **Cap:** none

### C5 Untrusted input blast radius — 0.07 (high)

Pod logs, events, exec output and resource fields are attacker-influenceable and are returned to the model as plain text with no provenance or untrusted marking. The default tool set gives a hijacked agent both exfiltration channels (a model-chosen Helm repository URL, creating pods that call out) and irreversible actions (forced deletes, Helm uninstall), with nothing in the server requiring a human. The opt-in read-only mode would remove most of the state-changing leg but is off by default.

- **S L1:** Tool outputs are raw kubectl stdout returned as plain text, with no source or untrusted flag; this is the absence of a mechanism, not an opt-in one. — [src/tools/kubectl-generic.ts:169-176](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L169-L176); [src/tools/kubectl-logs.ts:116-121](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-logs.ts#L116-L121) (verified)
  - *To reach the next level:* Structured outputs separating returned content from metadata, with provenance the host can act on.
- **C L0:** No untrusted source is distinguished: logs, events, exec output and resource fields all enter context the same way. — [src/tools/kubectl-logs.ts:116-121](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-logs.ts#L116-L121); searched `rg -n -i 'untrusted|provenance|prompt injection'` in `src/` → 1 hits (The single hit is a comment in kubectl-flags.ts explaining the flag denylist; no output marking exists.) (verified)
  - *To reach the next level:* Untrusted sources such as pod logs would need to be marked.
- **D L0:** There is no untrusted-input mechanism to turn on. — [src/index.ts:84-104](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L84-L104) (verified)
  - *To reach the next level:* No mechanism on by default.
- **B L0:** A hijacked session can read cluster data and exfiltrate it (helm repo add to any URL, pods with egress) and force-delete resources with no human step in the server. — [src/tools/helm-operations.ts:342-346](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/helm-operations.ts#L342-L346); [src/tools/kubectl-delete.ts:166-169](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-delete.ts#L166-L169) (verified)
  - *To reach the next level:* No leg of the Rule of Two is dropped by default.
- **Cap:** C5-WORSTCASE — B is L0: leak plus irreversible action, unattended, in the default configuration.

### C6 Memory, context & configuration integrity — 0.10 (medium)

The server has no memory store, but the model can persistently rewrite the operator's kubeconfig, which the server and the operator's own kubectl load on every later run. kubectl_context's set operation switches the current context, and it is not the only kubeconfig-writing path. A single injected instruction can therefore leave behind a configuration that points future sessions, and the operator's own terminal, at a different cluster.

- **S L0:** kubectl_context set runs 'kubectl config use-context', and it is not the only kubeconfig-writing path, so the model can rewrite the kubeconfig that every later session loads; kubectl's persistence of these writes is inferred from its documented behaviour. — [src/tools/kubectl-context.ts:300](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-context.ts#L300); [src/tools/kubectl-generic.ts:98-99](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L98-L99) (inferred)
  - *To reach the next level:* Writes to security-relevant configuration need a gate (human approval or validation).
- **C L0:** No path that writes the kubeconfig is controlled. — [src/tools/kubectl-generic.ts:98-99](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L98-L99) (verified)
  - *To reach the next level:* At least one persistence path would need to be controlled.
- **D L1:** The kubeconfig is per OS user (file permissions), but it is shared by every MCP session and by the operator's own kubectl. — [src/utils/kubernetes-manager.ts:79-85](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/utils/kubernetes-manager.ts#L79-L85) (verified)
  - *To reach the next level:* Per-session isolation of configuration enforced in code; capped one level above S.
- **B L1:** A poisoned kubeconfig persists across the user's sessions and changes which cluster or endpoint later tool calls (and the operator's own commands) hit. — [src/utils/kubernetes-manager.ts:79-85](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/utils/kubernetes-manager.ts#L79-L85); [src/tools/kubectl-context.ts:300](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-context.ts#L300) (inferred)
  - *To reach the next level:* Poisoned configuration should be session-scoped or easily inspected and reverted by the server.
- **Cap:** none
- **Notes:** No workspace files (.env, project config) are loaded from the working directory; C6-REPOCONFIG does not apply.

### C7 Third-party extensions — 1.00 (high)

The server loads no plugins, MCP servers or downloaded code into its own process: it only invokes the kubectl and helm binaries already installed by the operator. Helm charts installed from model-chosen repositories run in the cluster rather than in the server and are covered under tool scoping and untrusted-input blast radius.

- **Structural absence:** searched `rg -n 'import\(|require\('` in `src/` → 0 hits (No dynamic code loading.); searched `rg -n -i 'npx|npm install|pip install|plugin|loadPlugin'` in `src/` → 0 hits (No package install or plugin loader.)
- **Notes:** kubectl_generic's free 'command' lets the model invoke any kubectl plugin (kubectl-<name>) the operator has installed on PATH; these are the operator's own tools, not extensions the server loads. Helm charts are pulled unpinned (no version parameter) from any repo URL; scored in C3/C5.

### C8 Secrets & sensitive-data protection — 0.25 (high)

Credentials come from the operator's kubeconfig or environment variables; inline kubeconfigs are written to a 0600 temp file. kubectl_get masks the values of Secret objects by default, but this is the only protected path: it can be turned off with MASK_SECRETS=false, it is not a complete boundary, and kubectl_generic can fetch Secrets or run 'config view --raw' to return the full kubeconfig, tokens and keys included, straight to the model. Telemetry is off by default and records only tool names and argument keys.

- **S L1:** Secret data/stringData masking in kubectl_get only, and not a complete boundary there; credentials come from env vars or the kubeconfig. — [src/tools/kubectl-get.ts:178-185](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-get.ts#L178-L185); [src/tools/kubectl-get.ts:472](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-get.ts#L472) (verified)
  - *To reach the next level:* Masking on all model-bound paths (generic kubectl output, exec output) and redaction in logs.
- **C L1:** Only kubectl_get output is masked; kubectl_generic output, exec_in_pod output, logs and error messages are returned unfiltered, and subprocesses get the full environment. — [src/tools/kubectl-generic.ts:169-176](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L169-L176); [src/tools/kubectl-generic.ts:163-167](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L163-L167); [src/tools/kubectl-generic.ts:162](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L162) (verified)
  - *To reach the next level:* Logs and transcripts plus at least the main model-bound paths protected.
- **D L2:** Telemetry is opt-in and content-free; masking defaults on but MASK_SECRETS=false disables it. — [src/config/telemetry-config.ts:76-86](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/config/telemetry-config.ts#L76-L86); [src/tools/kubectl-get.ts:178-185](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-get.ts#L178-L185) (verified)
  - *To reach the next level:* Redaction that cannot be disabled.
- **B L0:** The model can obtain the operator's long-lived, high-privilege kubeconfig credentials (kubectl_generic config view --raw, or exec cat of service-account tokens). — [src/tools/kubectl-generic.ts:98-99](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L98-L99); [src/tools/exec_in_pod.ts:105-124](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/exec_in_pod.ts#L105-L124) (verified)
  - *To reach the next level:* Credentials reachable by the model are not scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.28 (medium)

By default the server keeps no record of what it did: only kubectl_generic prints the command it runs to stderr, and nothing is written to a file the server owns. An opt-in OpenTelemetry integration wraps every tool call in a span, but it records only the tool name, argument names, context, namespace and resource type, not the commands or manifests, and it is off unless both ENABLE_TELEMETRY and an OTLP endpoint are set.

- **default configuration** (default; raw 0.20 → 0.20)
  - **S L1:** An unstructured 'Executing: kubectl ...' line to stderr for kubectl_generic only. — [src/tools/kubectl-generic.ts:162](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L162) (verified)
    - *To reach the next level:* Structured record of every tool call with arguments, status and timestamps.
  - **C L1:** Only one tool logs what it runs; exec_in_pod, delete, apply and helm calls are not recorded. — [src/tools/kubectl-generic.ts:162](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L162); [src/tools/exec_in_pod.ts:105-124](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/exec_in_pod.ts#L105-L124) (verified)
    - *To reach the next level:* Every built-in tool call recorded.
  - **D L1:** The stderr line is on by default but is only as durable as the host's capture of the server's stderr. — [src/tools/kubectl-generic.ts:162](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L162); [src/index.ts:614-629](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L614-L629) (verified)
    - *To reach the next level:* Record written outside the server's process to durable storage by default.
  - **B L0:** There is no durable record to fail; actions proceed and nothing is retained by the server. — [src/tools/kubectl-generic.ts:162](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-generic.ts#L162) (verified)
    - *To reach the next level:* Errors surfaced and records flushed per action.
- **opt-in OpenTelemetry tracing** (alt; raw 0.28, cap G1 → 0.28) ← counted
  - **S L1:** Spans carry tool name, argument keys, context/namespace/resourceType and status, but not the command, manifest or exec argv. — [src/middleware/telemetry-middleware.ts:50-64](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/middleware/telemetry-middleware.ts#L50-L64) (verified)
    - *To reach the next level:* Spans would need the full arguments to reconstruct what ran.
  - **C L2:** withTelemetry wraps the single CallTool handler, so every tool call, including denials from isToolAllowed, gets a span. — [src/index.ts:247-249](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L247-L249); [src/index.ts:256-261](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L256-L261) (verified)
    - *To reach the next level:* Coverage is capped one level above the record's strength.
  - **D L0:** Enabled only when ENABLE_TELEMETRY=true and OTEL_EXPORTER_OTLP_ENDPOINT are both set. — [src/config/telemetry-config.ts:76-86](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/config/telemetry-config.ts#L76-L86) (verified)
    - *To reach the next level:* Not on by default.
  - **B L1:** Spans are exported best-effort by the SDK; export failures do not affect the action. — [src/config/telemetry-config.ts:76-86](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/config/telemetry-config.ts#L76-L86) (inferred)
    - *To reach the next level:* Records flushed per action with surfaced errors.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch — 0.33 (high)

Every kubectl and helm call has a default 1 MB output cap, and exec_in_pod, Helm and node operations have timeouts. But most kubectl calls (get, apply, delete, generic, logs with follow) have no timeout and run synchronously, the model can raise exec_in_pod and rollout timeouts itself, and there are no rate limits. Port-forward processes the server spawns are not stopped by cleanup or on shutdown.

- **S L2:** Server-enforced output cap (maxBuffer, 1 MB default) on every execFileSyncSafe call and timeouts on exec_in_pod, helm and node operations. — [src/config/max-buffer.ts:1-3](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/config/max-buffer.ts#L1-L3); [src/tools/exec_in_pod.ts:117](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/exec_in_pod.ts#L117); [src/tools/helm-operations.ts:166-172](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/helm-operations.ts#L166-L172) (verified)
  - *To reach the next level:* Caps on every operation (timeouts on get/apply/delete/generic/logs) plus rate or concurrency limits.
- **C L1:** The output cap applies to all synchronous kubectl/helm calls, but kubectl_get, kubectl_delete, kubectl_generic and kubectl_logs (including --follow) run without a timeout and port_forward has no cap. — [src/tools/kubectl-get.ts:169-173](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-get.ts#L169-L173); [src/tools/kubectl-logs.ts:116-120](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-logs.ts#L116-L120); [src/tools/kubectl-logs.ts:299-301](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/kubectl-logs.ts#L299-L301) (verified)
  - *To reach the next level:* Timeouts on every tool, not only exec, helm and node operations.
- **D L1:** Defaults exist, but exec_in_pod's timeout is a model-supplied argument with no ceiling. — [src/tools/exec_in_pod.ts:117](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/exec_in_pod.ts#L117) (verified)
  - *To reach the next level:* Limits the model cannot raise.
- **B L1:** Stopping the server leaves spawned kubectl port-forward processes running; cleanup() stops watches and tracked resources but not port-forwards. — [src/tools/port_forward.ts:15-17](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/tools/port_forward.ts#L15-L17); [src/utils/kubernetes-manager.ts:301-321](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/utils/kubernetes-manager.ts#L301-L321); [src/index.ts:631-637](https://github.com/Flux159/mcp-server-kubernetes/blob/3d71add204014dcceddebe53e9695ed7d8a6d129/src/index.ts#L631-L637) (verified)
  - *To reach the next level:* Stop should end all spawned processes.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: pod logs, events and exec output returned as plain text (src/tools/kubectl-logs.ts:116-121, src/tools/exec_in_pod.ts:105-124) · [B] sensitive data/systems: operator kubeconfig credentials and cluster Secrets via kubectl_generic (src/tools/kubectl-generic.ts:98-145) · [C] state change / egress: kubectl_delete --force, helm repo add to any URL (src/tools/kubectl-delete.ts:166-169, src/tools/helm-operations.ts:342-346) · Same default session? Yes

## Highest-impact improvements
1. Restrict kubectl_generic to an allowlist of verbs and subcommands. — C3 S L1→L2, +0.075 before caps (Playbook 3)
2. Fix tool annotations and read-only mode coverage, and add hints to kubectl_create and port_forward. — C2 S L1→L2, +0.075 before caps (Playbook 5)
3. Ship read-only (or non-destructive) mode as the default and require an explicit env var to enable write, exec and generic tools. — C3 D L2→L3, +0.050 before caps (Playbook 3)
4. Log every tool call with its full argv, status and timestamp to stderr as structured JSON by default. — C9 S L1→L2, +0.075 before caps (Playbook 1, step 3)
5. Cap model-supplied timeouts and add default timeouts to every kubectl call, and kill port-forward children on cleanup and shutdown. — C10 D L1→L2, +0.050 before caps (Playbook 3, step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- kubectl behaviour relied on (use-context persisting to the kubeconfig, config view --raw) is inferred from kubectl's documentation, not observed.
- Scored the stdio mode the README leads with; the Helm chart and SSE/Streamable HTTP transports were reviewed only for footnotes.
- Tests, CI workflows and the mcpb/gemini extension manifests were not examined in depth.
- No reviewer-injection text was found in README.md, AGENTS.md, CLAUDE.md or SECURITY.md.
