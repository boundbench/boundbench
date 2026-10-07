# Defense-in-Depth Score: Robusta

**Repo:** https://github.com/robusta-dev/robusta · **Commit:** `e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c` · **Reviewed:** 2026-10-03
**What it is:** Kubernetes Prometheus alert enrichment and automatic remediation platform with AI investigation via HolmesGPT
**Category:** Infrastructure & Ops
**Scored configuration:** Helm chart helm/robusta with shipped values.yaml defaults: cluster-wide runner ClusterRole, HolmesGPT disabled, no sinks, no extra playbook repos, in-cluster runner HTTP API on port 80.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication opt-in

## Score: 1.5 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L0 | L1 | L0 | L0 | 0.07 | — | **0.07** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | C4-HOSTROOT | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C7 | Third-party extensions | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C9 | Audit & traceability | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |


Robusta's in-cluster runner holds an admin-equivalent service account and exposes an unauthenticated HTTP endpoint (/api/trigger) that runs any registered action, including arbitrary bash in a privileged hostPID pod on any node and arbitrary kubectl commands. Any pod that can reach the runner service can therefore get node root and read the output back, and the chart ships no NetworkPolicy or approval step to stop it. The Robusta relay path is properly signed, and the AI component (HolmesGPT) is off by default and lives in a separate service, but the default runner is unsafe as shipped unless the operator locks down network access and RBAC.

## Critical gaps
- Unauthenticated /api/trigger runs any action, including privileged node bash, and returns its output to the caller. (ASI01, ASI02, LLM01; C5) — [src/robusta/runner/web.py:142-159](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/runner/web.py#L142-L159); [src/robusta/core/playbooks/playbooks_event_handler_impl.py:111-114](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/core/playbooks/playbooks_event_handler_impl.py#L111-L114); [src/robusta/integrations/kubernetes/custom_models.py:262-273](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/integrations/kubernetes/custom_models.py#L262-L273)
- The default node command-execution path creates privileged, hostPID pods with SYS_ADMIN (C4-HOSTROOT), giving host-equivalent reach. (ASI05, T11; C4) — [src/robusta/integrations/kubernetes/custom_models.py:262-273](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/integrations/kubernetes/custom_models.py#L262-L273); [src/robusta/integrations/kubernetes/custom_models.py:263](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/integrations/kubernetes/custom_models.py#L263)
- The runner's default ClusterRole can delete/patch pods and deployments cluster-wide and create pods with exec in its namespace, an admin-equivalent blast radius. (ASI03, T3; C1) — [helm/robusta/templates/runner-service-account.yaml:77-89](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/templates/runner-service-account.yaml#L77-L89); [helm/robusta/templates/runner-service-account.yaml:140-146](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/templates/runner-service-account.yaml#L140-L146); [helm/robusta/templates/runner-service-account.yaml:559-575](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/templates/runner-service-account.yaml#L559-L575)

## Criterion details

### C1 Identity & least privilege — 0.15 (high)

The runner uses its own Kubernetes service account, but the default chart binds it cluster-wide to a role that can delete and patch pods, patch and delete Deployments, patch nodes and evict pods everywhere, and create pods (including privileged ones) and exec into them in its own namespace, which together amount to cluster-admin-equivalent power. Requests that arrive through the Robusta relay are authenticated (HMAC signature, RSA partial keys, or a light-action allowlist), but the in-cluster HTTP endpoint /api/trigger runs any registered action with no authentication at all. The pods the runner spawns (kubectl jobs, privileged node debuggers) reuse the same service account. A namespace-scoped or read-only role exists only as opt-in Helm values.

- **S L1:** A dedicated runner ServiceAccount is bound to a broad ClusterRole (delete/patch pods, patch/delete deployments, patch nodes, evictions) plus a local Role that can create pods and pods/exec, which is admin-equivalent. — [helm/robusta/templates/runner-service-account.yaml:77-89](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/templates/runner-service-account.yaml#L77-L89); [helm/robusta/templates/runner-service-account.yaml:140-146](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/templates/runner-service-account.yaml#L140-L146); [helm/robusta/templates/runner-service-account.yaml:559-575](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/templates/runner-service-account.yaml#L559-L575) (verified)
  - *To reach the next level:* Role-scoped identity without pod-creation/patch-deployment powers that let it reach any namespace or node (no privileged pod creation, no patch on workloads cluster-wide).
- **C L1:** Relay requests pass signature/RSA/light-action checks, but /api/trigger calls run_external_action for any action with no authorization, and spawned kubectl/debugger pods run as the runner SA. — [src/robusta/integrations/receiver.py:333-338](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/integrations/receiver.py#L333-L338); [src/robusta/runner/web.py:142-159](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/runner/web.py#L142-L159); searched `rg -n 'before_request|login_required|Authorization'` in `src/robusta/runner` → 0 hits (No authentication middleware on the runner's Flask app.); [playbooks/robusta_playbooks/kubectl_enrichments.py:44](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/kubectl_enrichments.py#L44) (verified)
  - *To reach the next level:* Every action path, including the in-cluster HTTP API, must pass the same authorization check before running.
- **D L0:** The default chart renders a ClusterRole/ClusterRoleBinding; namespace-scoped and read-only roles are opt-in values. — [helm/robusta/values.yaml:772](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/values.yaml#L772); [helm/robusta/values.yaml:759](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/values.yaml#L759); [helm/robusta/templates/runner-service-account.yaml:6](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/templates/runner-service-account.yaml#L6) (verified)
  - *To reach the next level:* Ship a near-minimal (read-only or namespace-scoped) role by default and require explicit elevation for write/pod-creation verbs.
- **B L0:** A caller who reaches the runner gets cluster-wide pod/workload deletion and patching and, through privileged hostPID debugger pods, root on any node. — [helm/robusta/templates/runner-service-account.yaml:77-89](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/templates/runner-service-account.yaml#L77-L89); [helm/robusta/templates/runner-service-account.yaml:140-146](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/templates/runner-service-account.yaml#L140-L146); [src/robusta/integrations/kubernetes/custom_models.py:262-273](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/integrations/kubernetes/custom_models.py#L262-L273) (verified)
  - *To reach the next level:* Blast radius limited to one namespace with mostly-read access and no node-level execution.
- **Cap:** none
- **Notes:** D is L0 not because the role is opt-in (G1 does not apply: the dedicated SA and its ClusterRole are on by default) but because the default role is admin-equivalent; narrower roles (namespaceScoped, overrideClusterRoles) are opt-in hardening.

### C2 Approval gates — 0.00 (high)

There is no human approval step in front of the runner's consequential actions. Operator-configured playbooks run automatically on alerts and Kubernetes events, the in-cluster /api/trigger endpoint runs any action immediately, and relay/Slack requests execute as soon as their signature or light-action check passes. Slack buttons carry pre-signed action requests that skip the timestamp check, so anyone who can click the button in the channel triggers the action. HolmesGPT's tool-approval flag is a separate, opt-in feature of the AI service and defaults to off.

- **S L0:** No approval mechanism exists in the runner's action executor; actions run directly once requested or triggered. — [src/robusta/core/playbooks/playbooks_event_handler_impl.py:240-244](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/core/playbooks/playbooks_event_handler_impl.py#L240-L244); searched `rg -n -i 'approv'` in `src/robusta playbooks/robusta_playbooks` → 9 hits (All hits are the HolmesGPT chat tool-approval fields (enable_tool_approval default False) for the external AI service, not a gate on runner actions.) (verified)
  - *To reach the next level:* Per-call human approval showing the exact action and parameters before consequential actions (delete_pod, drain, node_bash_enricher, kubectl_command).
- **C L0:** The most powerful paths (node_bash_enricher, kubectl_command, delete_pod via /api/trigger) reach execution without any gate. — [src/robusta/runner/web.py:142-159](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/runner/web.py#L142-L159); [playbooks/robusta_playbooks/bash_enrichments.py:26-38](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/bash_enrichments.py#L26-L38) (verified)
  - *To reach the next level:* Every consequential action path must traverse a human gate.
- **D L0:** No gate exists, so there is nothing on by default; Holmes tool approval defaults to False. — [src/robusta/core/model/base_params.py:207](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/core/model/base_params.py#L207) (verified)
  - *To reach the next level:* An approval gate on by default for destructive actions.
- **B L0:** Unapproved actions include pod deletion, node drain, arbitrary kubectl and root shell on nodes, with no undo. — [playbooks/robusta_playbooks/node_actions.py:50-56](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/node_actions.py#L50-L56); [playbooks/robusta_playbooks/bash_enrichments.py:26-38](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/bash_enrichments.py#L26-L38) (verified)
  - *To reach the next level:* Reversible or previewed actions only, with rate limits on consequential actions.
- **Cap:** none

### C3 Tool & action scoping — 0.07 (high)

Actions take typed parameters, but the most powerful ones are raw passthroughs: node_bash_enricher and pod_bash_enricher run any bash string, kubectl_command runs any shell string through /bin/sh -c, http_stress_test hits any URL, and ask_holmes accepts an arbitrary Holmes URL. Every bundled action is registered and callable by default, and the only narrowing is the relay's light-action name list, which itself includes destructive actions such as delete_pod and drain and does not cover the in-cluster HTTP API.

- **S L0:** bash_command and kubectl command strings are passed straight to a shell; URLs are not validated. — [playbooks/robusta_playbooks/bash_enrichments.py:19](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/bash_enrichments.py#L19); [playbooks/robusta_playbooks/kubectl_enrichments.py:50-51](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/kubectl_enrichments.py#L50-L51); [playbooks/robusta_playbooks/stress_tests.py:30](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/stress_tests.py#L30) (verified)
  - *To reach the next level:* Allowlist validation of commands, hosts and targets in code, replacing raw shell with narrow actions.
- **C L1:** Pydantic parameter classes give type checks, and the relay path checks action names against lightActions, but no action validates its arguments against bounds. — [src/robusta/integrations/receiver.py:339-350](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/integrations/receiver.py#L339-L350); [src/robusta/core/model/base_params.py:313](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/core/model/base_params.py#L313) (verified)
  - *To reach the next level:* Most built-in actions validating arguments against allowlists/bounds.
- **D L0:** All default playbook actions (including node_bash_enricher and kubectl_command) are registered and callable via /api/trigger; the light-action list includes delete_pod and drain. — [src/robusta/runner/config_loader.py:227-229](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/runner/config_loader.py#L227-L229); [helm/robusta/values.yaml:79](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/values.yaml#L79); [helm/robusta/values.yaml:114](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/values.yaml#L114) (verified)
  - *To reach the next level:* A read-only action set by default with write/exec actions explicitly enabled.
- **B L0:** A misused action can run any command on any node or delete workloads anywhere in the cluster. — [src/robusta/integrations/kubernetes/custom_models.py:262-273](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/integrations/kubernetes/custom_models.py#L262-L273); [helm/robusta/templates/runner-service-account.yaml:77-89](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/templates/runner-service-account.yaml#L77-L89) (verified)
  - *To reach the next level:* Actions scoped to a namespace and bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

Caller-supplied commands are executed with no meaningful isolation. node_bash_enricher creates a privileged pod with hostPID, SYS_ADMIN and the runner's service account on the target node and executes the command there, which is root on the node. pod_bash_enricher executes inside the target workload's own container, and kubectl_command runs the string in a stock container that holds the runner's cluster-wide token. None of this is gated or sandboxed, and the chart does not set a restricted policy for these pods.

- **S L0:** The node execution path is a privileged hostPID container with SYS_ADMIN, which is no isolation from the host. — [src/robusta/integrations/kubernetes/custom_models.py:262-273](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/integrations/kubernetes/custom_models.py#L262-L273); [src/robusta/integrations/kubernetes/custom_models.py:263](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/integrations/kubernetes/custom_models.py#L263) (verified)
  - *To reach the next level:* A hardened, non-root, capability-dropped sandbox with no host namespaces.
- **C L0:** No execution path is sandboxed: node bash (privileged), pod bash (inside the workload) and kubectl_command (runner SA token) all run unconfined. — [playbooks/robusta_playbooks/bash_enrichments.py:26-38](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/bash_enrichments.py#L26-L38); [playbooks/robusta_playbooks/bash_enrichments.py:19](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/bash_enrichments.py#L19); [playbooks/robusta_playbooks/kubectl_enrichments.py:44](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/kubectl_enrichments.py#L44) (verified)
  - *To reach the next level:* Every command-execution path running inside an isolation boundary.
- **D L0:** There is no sandbox to enable; privileged debugger pods are the built-in behaviour. — [src/robusta/integrations/kubernetes/custom_models.py:273](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/integrations/kubernetes/custom_models.py#L273) (verified)
  - *To reach the next level:* A sandbox on by default whose policy the caller cannot change.
- **B L0:** Commands run with host PID namespace, privileged mode and the runner's service-account token mounted. — [src/robusta/integrations/kubernetes/custom_models.py:262-273](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/integrations/kubernetes/custom_models.py#L262-L273); [helm/robusta/values.yaml:47](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/values.yaml#L47) (verified)
  - *To reach the next level:* Workload-only reach, no secrets in the environment, restricted egress.
- **Cap:** C4-HOSTROOT — The default command-execution path for nodes creates a privileged, hostPID pod (custom_models.py:262-273).

### C5 Untrusted input blast radius — 0.00 (high)

The runner acts on input it cannot authenticate: Alertmanager webhooks, Kubernetes events and the /api/trigger endpoint all accept unauthenticated POSTs from anywhere inside the cluster. The chart ships no NetworkPolicy, and the server binds to 0.0.0.0. Through /api/trigger, any pod in the cluster can run node_bash_enricher or kubectl_command and, with sync_response, get the output back in the HTTP response. That combines data exfiltration with irreversible actions, and no human is involved. Nothing in the code tracks where an input came from or how far it is trusted. When HolmesGPT is enabled, alert labels are also pasted into the AI prompt.

- **S L0:** No structural limit: unauthenticated input selects actions, targets and parameters. — [src/robusta/runner/web.py:142-159](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/runner/web.py#L142-L159); searched `rg -n 'before_request|login_required|Authorization'` in `src/robusta/runner` → 0 hits (No authentication middleware on the runner's Flask app.) (verified)
  - *To reach the next level:* Some dangerous capabilities requiring human approval once untrusted input is involved.
- **C L0:** Alerts, k8s events and HTTP triggers are not distinguished from operator intent. — [src/robusta/runner/web.py:87-107](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/runner/web.py#L87-L107); searched `rg -n -i 'untrusted|provenance'` in `src/robusta` → 0 hits (No provenance or taint tracking anywhere in the runner.) (verified)
  - *To reach the next level:* Distinguishing untrusted sources from operator configuration and limiting what they can trigger.
- **D L0:** No control exists; the endpoint is documented as open in-cluster by design. — [docs/playbook-reference/triggers/webhook.rst:7-15](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/docs/playbook-reference/triggers/webhook.rst#L7-L15); searched `rg -n -i 'kind: NetworkPolicy'` in `helm/robusta/templates` → 0 hits (The chart ships no NetworkPolicy restricting who can reach the runner service.) (verified)
  - *To reach the next level:* A control on by default (authenticated triggers, NetworkPolicy).
- **B L0:** An in-cluster attacker can run root commands on nodes and read the output back via sync_response, and delete workloads, unattended. — [src/robusta/core/playbooks/playbooks_event_handler_impl.py:111-114](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/core/playbooks/playbooks_event_handler_impl.py#L111-L114); [src/robusta/integrations/kubernetes/custom_models.py:262-273](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/integrations/kubernetes/custom_models.py#L262-L273); [src/robusta/core/model/env_vars.py:101](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/core/model/env_vars.py#L101) (verified)
  - *To reach the next level:* Both exfiltration and irreversible actions requiring human approval.
- **Cap:** C5-WORSTCASE — B is L0: unauthenticated callers can both read data back and take irreversible actions with no human involved.

### C6 Memory, context & configuration integrity — 0.25 (high)

The runner has no AI memory, and it does not load configuration from any workspace. Its playbook configuration comes from a Kubernetes Secret that its own role cannot update. However, the role can patch Deployments cluster-wide, including the runner's own, and can create ConfigMaps in its namespace. A caller of /api/trigger can therefore use kubectl_command to rewrite the runner's environment or mount a different playbook config. That change survives restarts and can trigger later actions. No code checks or reverts changes to the runner's own configuration.

- **S L1:** Playbook config is an operator-owned Secret the runner cannot update, but the runner's own Deployment (env, volumes, PLAYBOOKS_CONFIG_FILE_PATH) is writable through its patch-deployments permission. — [helm/robusta/templates/runner.yaml:108-109](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/templates/runner.yaml#L108-L109); [helm/robusta/templates/runner-service-account.yaml:140-146](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/templates/runner-service-account.yaml#L140-L146); [helm/robusta/templates/runner-service-account.yaml:559-567](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/templates/runner-service-account.yaml#L559-L567) (verified)
  - *To reach the next level:* Security-relevant runtime configuration that the runner's own tools cannot modify.
- **C L1:** Only the playbooks Secret is out of the runner's reach; Deployment spec and ConfigMaps (incl. persistent_data configmaps) are not. — [helm/robusta/templates/runner-service-account.yaml:77-89](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/templates/runner-service-account.yaml#L77-L89) (verified)
  - *To reach the next level:* All persistent configuration paths protected from the runner's own actions.
- **D L1:** Single-tenant runner with no namespacing of persistent state; its own actions can rewrite its deployment. — [helm/robusta/templates/runner-service-account.yaml:140-146](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/templates/runner-service-account.yaml#L140-L146) (verified)
  - *To reach the next level:* Persistent state the runner's actions cannot rewrite.
- **B L1:** A tampered deployment/config persists across restarts and drives future automatic actions. — [src/robusta/runner/config_loader.py:205-213](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/runner/config_loader.py#L205-L213) (verified)
  - *To reach the next level:* Persistent changes only after human review, or easily detected and reverted.
- **Cap:** none

### C7 Third-party extensions — 0.25 (high)

By default the runner loads no third-party playbook packages. When an operator adds playbookRepos, they are pip-installed from a git branch or tarball URL with no hash check and imported into the runner's process. Built-in actions pull third-party container images at run time: bitnami/kubectl:latest with imagePullPolicy Always, and the untagged williamyeh/hey. The kubectl image runs with the runner's service account, so a compromised upstream image inherits the runner's cluster-wide authority.

- **S L1:** Runtime images are mostly tag-pinned but bitnami/kubectl:latest and untagged williamyeh/hey are unpinned; playbook repos have no integrity check. — [playbooks/robusta_playbooks/kubectl_enrichments.py:17](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/kubectl_enrichments.py#L17); [src/robusta/runner/config_loader.py:140-146](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/runner/config_loader.py#L140-L146); searched `rg -n 'sha256:|@sha256'` in `src/robusta playbooks/robusta_playbooks` → 0 hits (No image digest pinning anywhere.) (verified)
  - *To reach the next level:* Version-pinned images and packages for every third-party source.
- **C L1:** Neither images nor playbook repos are verified; only some images carry fixed tags. — [src/robusta/runner/config_loader.py:183](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/runner/config_loader.py#L183) (verified)
  - *To reach the next level:* Most extension types pinned.
- **D L1:** No playbook repos by default, but built-in actions pull and run third-party images on invocation with no consent step. — [helm/robusta/values.yaml:10](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/values.yaml#L10); [playbooks/robusta_playbooks/kubectl_enrichments.py:49](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/kubectl_enrichments.py#L49) (verified)
  - *To reach the next level:* Nothing third-party run by default; adding one shows exactly what will run.
- **B L1:** The kubectl image runs as a separate pod with the runner's service-account token; opt-in playbook repos run in-process. — [playbooks/robusta_playbooks/kubectl_enrichments.py:44](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/kubectl_enrichments.py#L44) (verified)
  - *To reach the next level:* Third-party code in a separate process with only its own scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.20 (high)

Secrets are stored in a Kubernetes Secret and passed to the runner as environment variables or a mounted config. Action parameters are partly masked in logs, and a few sinks use pydantic SecretStr, but the Slack and PagerDuty keys are plain strings. Basic telemetry is on by default and sends counts and a hashed account ID. Sentry error reporting is opt-in. The runner's service-account token, which is admin-equivalent, is mounted by default and passed to every pod the runner spawns. The signing key that authorizes relay actions is a long-lived shared secret.

- **S L1:** Secrets come from a k8s Secret/env; masking exists only for action-param log lines (safe_str) and some sink fields. — [src/robusta/core/playbooks/playbook_utils.py:46-56](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/core/playbooks/playbook_utils.py#L46-L56); [src/robusta/core/sinks/slack/slack_sink_params.py:11](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/core/sinks/slack/slack_sink_params.py#L11) (verified)
  - *To reach the next level:* Type-level masking and log filters on all main paths, including the Slack/PagerDuty keys.
- **C L1:** Masking covers action-param logging; raw trace logging of requests and subprocess/pod token propagation are unprotected. — [src/robusta/runner/web.py:176-184](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/runner/web.py#L176-L184) (verified)
  - *To reach the next level:* Logs and transcripts both protected.
- **D L1:** Content-free telemetry is on by default; raw request tracing is one env var away; Sentry opt-in. — [src/robusta/core/model/env_vars.py:73](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/core/model/env_vars.py#L73); [src/robusta/runner/telemetry.py:12-21](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/runner/telemetry.py#L12-L21); [helm/robusta/values.yaml:738](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/values.yaml#L738) (verified)
  - *To reach the next level:* Telemetry opt-in with redaction always on.
- **B L0:** Long-lived admin-equivalent SA token is automounted in the runner and in every spawned pod; signing_key authorizes any action via relay. — [helm/robusta/values.yaml:47](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/values.yaml#L47); [src/robusta/integrations/kubernetes/custom_models.py:262](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/integrations/kubernetes/custom_models.py#L262); [helm/robusta/values.yaml:62](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/helm/robusta/values.yaml#L62) (verified)
  - *To reach the next level:* Scoped keys rather than cluster-admin-equivalent long-lived tokens.
- **Cap:** none

### C9 Audit & traceability — 0.30 (high)

The runner does not keep a structured record of the actions it runs. Successful actions are not logged by the executor. Relay callbacks are logged only at debug level, and incoming /api/trigger requests are traced only if a debug env var is set. Failures and some individual actions (for example kubectl_command's description) are logged to stdout. Prometheus counters record counts and durations, and findings go to sinks when the operator has configured any. None of this records which caller requested an action.

- **S L1:** Unstructured stdout logs of errors and some actions; successful action invocations and their callers are not recorded. — [src/robusta/integrations/receiver.py:142](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/integrations/receiver.py#L142); [playbooks/robusta_playbooks/kubectl_enrichments.py:40](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/kubectl_enrichments.py#L40); searched `rg -n 'audit'` in `src/robusta/core/playbooks src/robusta/runner` → 0 hits (No audit module in the executor or runner.) (verified)
  - *To reach the next level:* A structured record of every action with arguments, caller and timestamp.
- **C L1:** Only errors and a few actions emit logs; relay and HTTP triggers are untraced by default. — [src/robusta/core/model/env_vars.py:94](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/core/model/env_vars.py#L94) (verified)
  - *To reach the next level:* All action paths recorded.
- **D L2:** Container stdout logs are on by default and collected outside the runner by the kubelet, but are written by the runner process itself. — [src/robusta/core/playbooks/playbooks_event_handler_impl.py:273-276](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/core/playbooks/playbooks_event_handler_impl.py#L273-L276) (verified)
  - *To reach the next level:* Records written by a component the runner cannot control.
- **B L1:** Logging is best-effort; nothing blocks an action if its record isn't written. — [src/robusta/core/playbooks/playbooks_event_handler_impl.py:240-244](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/core/playbooks/playbooks_event_handler_impl.py#L240-L244) (verified)
  - *To reach the next level:* Errors surfaced and records flushed per action.
- **Cap:** none

### C10 Limits & kill switch — 0.25 (high)

The runner bounds its intake with a 500-entry event queue, a fixed worker pool, and per-action timeouts such as the kubectl job wait. The caller sets the kubectl timeout, and it defaults to an hour. Per-trigger rate limits exist but are opt-in per playbook. There are no overall caps on actions, time, or spawned pods. Jobs the runner creates can keep running for up to 12 hours (ttlSecondsAfterFinished). Stopping the runner does not stop pods or jobs it already launched.

- **S L1:** Bounded queue and per-action timeouts only; no overall action/time limit. — [src/robusta/utils/task_queue.py:39](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/utils/task_queue.py#L39); [src/robusta/core/model/env_vars.py:44](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/src/robusta/core/model/env_vars.py#L44) (verified)
  - *To reach the next level:* Enforced iteration/step caps plus wall-clock limits on runs.
- **C L1:** Limits apply to intake; spawned jobs/pods are not counted. — [playbooks/robusta_playbooks/kubectl_enrichments.py:58-66](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/kubectl_enrichments.py#L58-L66) (verified)
  - *To reach the next level:* Spawned pods and jobs bounded by the same limits.
- **D L1:** The kubectl timeout defaults to 3600s and is set by the caller's parameters. — [playbooks/robusta_playbooks/kubectl_enrichments.py:29](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/kubectl_enrichments.py#L29) (verified)
  - *To reach the next level:* Sensible defaults the caller cannot raise.
- **B L1:** Jobs live up to 12h; stopping the runner leaves spawned pods/jobs running. — [playbooks/robusta_playbooks/kubectl_enrichments.py:63](https://github.com/robusta-dev/robusta/blob/e6b2b563a9c0f78a2c1bfa4c1f1aac60266f390c/playbooks/robusta_playbooks/kubectl_enrichments.py#L63) (verified)
  - *To reach the next level:* Stopping cancels in-flight spawned work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Unauthenticated in-cluster POSTs to /api/trigger and /api/alerts (src/robusta/runner/web.py:87, :142) · [B] sensitive data/systems: Cluster-wide pod/log reads and node root via privileged debugger pods (src/robusta/integrations/kubernetes/custom_models.py:262-273) · [C] state change / egress: Cluster-wide delete/patch of pods and deployments (helm/robusta/templates/runner-service-account.yaml:77-89, 140-146) and sync_response output (src/robusta/core/playbooks/playbooks_event_handler_impl.py:111-114) · Same default session? Yes

## Highest-impact improvements
1. Require the signing-key HMAC (or a bearer token) on /api/trigger and the other runner HTTP endpoints, and ship a NetworkPolicy allowing only Alertmanager/kubewatch. — C1 C L1→L2, +0.075 before caps (Playbook 4)
2. Default to the namespace-scoped role and make cluster-wide write verbs and privileged pod creation an explicit opt-in. — C1 D L0→L2, +0.100 before caps (Playbook 4)
3. Drop privileged/hostPID from debugger pods by default (or make node_bash_enricher opt-in) and run spawned pods with a separate minimal service account. — C4 S L0→L2, +0.150 before caps (Playbook 3)
4. Log every action invocation at INFO with action name, source (relay, slack, http, playbook), caller and timestamp. — C9 S L1→L2, +0.075 before caps (Playbook 1 step 3)
5. Pin bitnami/kubectl and williamyeh/hey to versions or digests instead of latest. — C7 S L1→L2, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The HolmesGPT AI agent loop lives in a separate repository and in the vendored chart charts/holmes-0.42.0.tgz, which were not reviewed; it is disabled by default (enableHolmesGPT: false), so this score covers the runner, its actions and the Helm chart.
- Whether privileged debugger pods are admitted depends on the cluster's Pod Security admission settings for the Robusta namespace, which the chart does not configure.
- Scored the raw chart defaults; the documented installer (robusta gen-config) typically adds a Robusta UI sink, signing key and platform playbooks, which add the authenticated relay path but do not change the unauthenticated in-cluster API.
- No reviewer-injection attempts were found; .claude/skills/patch-cves is a developer workflow skill.
