# Defense-in-Depth Score: kubectl-ai

**Repo:** https://github.com/GoogleCloudPlatform/kubectl-ai · **Commit:** `2c8ff822bacba3009f215df204610a29f5632454` · **Reviewed:** 2026-10-03
**What it is:** AI-powered Kubernetes assistant that translates intent into kubectl operations; MCP server/client
**Category:** Infrastructure & Ops
**Scored configuration:** Interactive terminal CLI with no flags: Gemini provider, the user's current kubeconfig, local (unsandboxed) executor, in-memory sessions, default trace file in /tmp.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents no · external communication no

## Score: 2.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L0 | L2 | 0.40 | G1 | **0.40** (alt) | Medium |
| C2 | Approval gates | L2 | L1 | L1 | L0 | 0.28 | G2 | **0.25** | High |
| C3 | Tool & action scoping | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C4 | Code-execution isolation | L2 | L2 | L0 | L0 | 0.30 | G1 | **0.30** (alt) | High |
| C5 | Untrusted input blast radius | L2 | L0 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | Medium |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | Medium |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C10 | Limits & kill switch | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |


kubectl-ai runs model-written bash and kubectl commands on your machine with your full kubeconfig and environment. Its main defence is a parser-based approval prompt for anything that is not a single read-only kubectl call. That allowlist is not a strict boundary, and the default-on trace does not protect sensitive data.

## Critical gaps
- All commands run with the operator's full kubeconfig authority and entire environment; no scoped identity in the default configuration. (ASI03, T3; C1) — [pkg/tools/kubectl_tool.go:121](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/kubectl_tool.go#L121); [pkg/tools/bash_tool.go:98](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/bash_tool.go#L98)
- Model commands run unsandboxed on the host by default; the opt-in k8s sandbox exports the host's full environment (API keys) into the pod. (ASI05, T11; C4) — [pkg/sandbox/local.go:48](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/sandbox/local.go#L48); [pkg/sandbox/kubernetes.go:61-63](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/sandbox/kubernetes.go#L61-L63)
- When MCP is enabled, the default server is an unpinned 'npx -y' package and servers run as the same user. (ASI04, T17; C7) — [pkg/mcp/default_config.yaml:3-6](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/mcp/default_config.yaml#L3-L6); [pkg/mcp/stdio_client.go:78](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/mcp/stdio_client.go#L78)

## Criterion details

### C1 Identity & least privilege — 0.40 (medium)

kubectl-ai runs every command with whatever credentials your kubeconfig holds, plus your entire shell environment, and never narrows them. There is no authorization layer between the model's requests and the cluster: the only control is the approval prompt scored under approval gates. An opt-in Kubernetes sandbox runs commands in a pod under a dedicated read-only service account defined in the shipped RBAC manifests, which is a real narrowing, but it is off by default and MCP servers still run on the host with full authority.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Commands run as the operator with the ambient kubeconfig and the full process environment; no scoped identity is created or requested. — [pkg/tools/kubectl_tool.go:121](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/kubectl_tool.go#L121); [pkg/tools/bash_tool.go:98](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/bash_tool.go#L98); [cmd/main.go:682-698](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L682-L698) (verified)
    - *To reach the next level:* Run model commands under a dedicated role-scoped identity (e.g. the sandbox service account) by default.
  - **C L0:** No authorization check exists on any tool path; bash, kubectl, custom and MCP tools all inherit the same ambient credentials. — [pkg/tools/kubectl_tool.go:121](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/kubectl_tool.go#L121); [pkg/tools/custom_tool.go:141](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/custom_tool.go#L141) (verified)
    - *To reach the next level:* Route every tool through one authorization layer that checks the request against a policy before credentials are used.
  - **D L0:** The default install uses the user's current kubeconfig context, commonly cluster-admin, with the local executor and no sandbox. — [cmd/main.go:208](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L208); [pkg/agent/conversation.go:271-273](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/agent/conversation.go#L271-L273) (verified)
    - *To reach the next level:* Default to a read-only identity and require explicit elevation for writes.
  - **B L0:** A hijacked agent holds the operator's full cluster authority and every credential in the environment (cloud CLIs, LLM keys). — [pkg/tools/bash_tool.go:98](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/bash_tool.go#L98); [cmd/main.go:686-687](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L686-L687) (verified)
    - *To reach the next level:* Limit the reachable authority to one namespace or read-only, with short-lived credentials.
- **opt-in --sandbox=k8s pod with the normal-user service account** (alt; raw 0.40, cap G1 → 0.40) ← counted
  - **S L2:** Sandbox pods run under the normal-user service account, which the shipped manifests bind to get/list/watch roles that exclude secrets. — [pkg/sandbox/kubernetes.go:327-334](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/sandbox/kubernetes.go#L327-L334); [k8s/sandbox/role.yaml:5](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/k8s/sandbox/role.yaml#L5) (verified)
    - *To reach the next level:* Use per-tool or per-request credentials, separating read and write authority.
  - **C L2:** bash, kubectl and custom tools execute inside the pod (the host KUBECONFIG path exported into the pod does not exist there, so kubectl is inferred to fall back to the pod's in-cluster service-account config); MCP servers and the agent process itself keep the host credentials. — [pkg/sandbox/kubernetes.go:61-63](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/sandbox/kubernetes.go#L61-L63); [pkg/mcp/stdio_client.go:78](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/mcp/stdio_client.go#L78) (inferred)
    - *To reach the next level:* Bring MCP/extension tool paths under the same scoped identity.
  - **D L0:** The sandbox and its RBAC are opt-in; the operator must apply k8s/sandbox manifests and pass --sandbox=k8s. — [cmd/main.go:208](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L208) (verified)
    - *To reach the next level:* Make the scoped identity the default execution path.
  - **B L2:** With the shipped roles the pod can read most cluster objects including configmaps and pod logs, but not secrets or writes. — [k8s/sandbox/cluster_role.yaml:9-11](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/k8s/sandbox/cluster_role.yaml#L9-L11) (verified)
    - *To reach the next level:* Narrow reads to the namespaces the task needs, with short-lived tokens.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C2 Approval gates — 0.25 (high)

A deterministic classifier decides which commands need your approval: anything it cannot prove to be a single read-only kubectl call is shown to you in full before it runs, including all bash, custom-tool and MCP calls, and compound shell commands are parsed rather than prefix-matched. The gap is the read-only allowlist itself, which is not a strict boundary. Every prompt also offers a session-wide 'don't ask me again'.

- **S L2:** Approval is requested for the whole pending batch, showing each exact command string, and always offers a session-wide allow-all; the prompt display is not a strict boundary. — [pkg/agent/conversation.go:757](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/agent/conversation.go#L757); [pkg/tools/tools.go:145-148](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/tools.go#L145-L148); [pkg/agent/conversation.go:785](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/agent/conversation.go#L785); [pkg/agent/conversation.go:1194](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/agent/conversation.go#L1194) (verified)
  - *To reach the next level:* Harden how the call is displayed and drop or time-bound the session-wide allow-all option.
- **C L1:** All non-kubectl, custom and MCP calls are gated, but the auto-approve allowlist is not a strict boundary. — [pkg/agent/conversation.go:723-730](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/agent/conversation.go#L723-L730); [pkg/tools/mcp_tool.go:103](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/mcp_tool.go#L103); [pkg/tools/custom_tool.go:159](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/custom_tool.go#L159); [pkg/tools/kubectl_filter.go:99-104](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/kubectl_filter.go#L99-L104) (verified)
  - *To reach the next level:* Make the auto-approved set a verified read-only allowlist.
- **D L1:** Approval is on by default, but the skipPermissions key in the auto-loaded user config disables it silently, and protection of the approval setting is not tamper-resistant. — [cmd/main.go:167](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L167); [cmd/main.go:87](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L87); [cmd/main.go:158-161](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L158-L161); [cmd/main.go:212-216](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L212-L216) (verified)
  - *To reach the next level:* Protect the agent's own config, and require the explicit --skip-permissions flag to disable approval.
- **B L0:** A bypassed or wrongly approved call can run arbitrary kubectl writes (delete, drain, apply) and shell commands against the cluster and host, with no checkpoint or undo. — [pkg/sandbox/local.go:48](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/sandbox/local.go#L48) (verified)
  - *To reach the next level:* Offer server-side dry-run previews and rollback points for cluster changes.
- **Cap:** G2 — Protection of the gate's approval setting is not tamper-resistant at runtime.

### C3 Tool & action scoping — 0.00 (high)

Both built-in tools take a free-form string and run it with bash -c, so the 'kubectl' tool is in practice a second shell tool. The only input checks block 'kubectl edit' and 'kubectl port-forward' substrings, to avoid interactive hangs rather than as a safety boundary. Nothing bounds namespaces, resources, paths or hosts, and the bash and kubectl tools are always enabled.

- **S L0:** Arguments are raw shell strings executed by bash -c; the only filter rejects interactive kubectl subcommands. — [pkg/tools/bash_tool.go:110-118](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/bash_tool.go#L110-L118); [pkg/sandbox/local.go:48](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/sandbox/local.go#L48) (verified)
  - *To reach the next level:* Replace raw strings with structured kubectl arguments validated against namespace/verb/resource allowlists.
- **C L0:** No tool validates its inputs against any bounds; custom and MCP tools pass arguments through unchanged. — [pkg/tools/bash_tool.go:110-118](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/bash_tool.go#L110-L118); [pkg/tools/custom_tool.go:135-150](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/custom_tool.go#L135-L150) (verified)
  - *To reach the next level:* Add a shared validation layer that every tool, including extensions, goes through.
- **D L0:** bash and kubectl (both exec-capable, write-capable and network-capable) are registered unconditionally for every session. — [pkg/agent/conversation.go:286-287](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/agent/conversation.go#L286-L287) (verified)
  - *To reach the next level:* Ship a read-only default tool set and require enabling write/exec tools.
- **B L0:** A misused tool can run any command on the host and any kubectl operation against any namespace in the current context. — [pkg/sandbox/local.go:48](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/sandbox/local.go#L48); [pkg/tools/kubectl_tool.go:121](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/kubectl_tool.go#L121) (verified)
  - *To reach the next level:* Scope tools to a namespace and bounded operations.
- **Cap:** none

### C4 Code-execution isolation — 0.30 (high)

By default every model command runs directly on your machine through bash, as your user, with your full environment and kubeconfig. The 'seatbelt' option on macOS is not a complete boundary. The opt-in Kubernetes sandbox runs commands in a stock pod (bitnami/kubectl:latest, no security context), which is basic container separation, but it exports the host's entire environment, including LLM API keys, into each command, and MCP servers still run on the host.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** The default local executor runs bash -c on the host as the current user. — [pkg/sandbox/local.go:48](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/sandbox/local.go#L48); [pkg/agent/conversation.go:271-273](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/agent/conversation.go#L271-L273) (verified)
    - *To reach the next level:* Execute model commands inside a hardened sandbox by default.
  - **C L0:** No path is sandboxed in the default configuration. — [pkg/agent/conversation.go:271-273](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/agent/conversation.go#L271-L273); [pkg/mcp/stdio_client.go:78](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/mcp/stdio_client.go#L78) (verified)
    - *To reach the next level:* Route every execution path (bash, kubectl, custom tools, MCP stdio servers) through the sandbox.
  - **D L0:** Sandboxing is off by default (sandbox defaults to the empty string, meaning the local executor). — [cmd/main.go:208](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L208) (verified)
    - *To reach the next level:* Turn the sandbox on by default.
  - **B L0:** Commands have host-equivalent reach: home directory, ~/.kube, cloud credentials and the full environment. — [pkg/tools/bash_tool.go:98](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/bash_tool.go#L98); [pkg/sandbox/local.go:48](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/sandbox/local.go#L48) (verified)
    - *To reach the next level:* Mount only a workspace, strip secrets from the environment and restrict egress.
- **opt-in --sandbox=k8s pod executor** (alt; raw 0.30, cap G1 → 0.30) ← counted
  - **S L2:** Commands run via exec in a stock pod with no securityContext, root/non-root left to the image and default capabilities. — [pkg/sandbox/kubernetes.go:327-334](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/sandbox/kubernetes.go#L327-L334); searched `rg -n -i 'securityContext|RunAsNonRoot|Capabilities|ReadOnlyRootFilesystem'` in `pkg/sandbox` → 0 hits (no pod hardening fields are set) (verified)
    - *To reach the next level:* Harden the pod: non-root, dropped capabilities, no-new-privileges, seccomp, read-only root filesystem.
  - **C L2:** bash, kubectl and custom tools use the pod executor and sandbox creation failure aborts the session, but MCP stdio servers still launch on the host. — [pkg/agent/conversation.go:242-262](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/agent/conversation.go#L242-L262); [pkg/mcp/stdio_client.go:78](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/mcp/stdio_client.go#L78) (verified)
    - *To reach the next level:* Run MCP stdio servers and every spawned process inside the sandbox.
  - **D L0:** The sandbox is off unless --sandbox=k8s is passed. — [cmd/main.go:208](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L208) (verified)
    - *To reach the next level:* Enable it by default with an explicit operator escape hatch.
  - **B L0:** Every command is prefixed with exports of the host's full environment, so LLM API keys and any cloud credentials in env are present inside the pod, which also has unrestricted network. — [pkg/sandbox/kubernetes.go:61-63](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/sandbox/kubernetes.go#L61-L63); [pkg/tools/bash_tool.go:98](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/bash_tool.go#L98) (verified)
    - *To reach the next level:* Pass no host environment into the sandbox and restrict egress.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.25 (medium)

kubectl-ai reads cluster content that any workload owner can control (pod logs, events, annotations, configmaps) and feeds it back to the model as ordinary tool results. Nothing tracks that content as untrusted. The approval gate does stop most shell egress and kubectl writes, and compound commands are deliberately sent for approval to block exfiltration. But a hijacked agent can still read secrets with an auto-approved 'kubectl get secret', and the gate does not cover every path, so it can exfiltrate data and delete resources without a prompt.

- **S L2:** Writes and non-kubectl egress always need approval (composite commands are deliberately forced to 'unknown' to resist exfiltration), but this is not tied to untrusted content and kubectl-read egress is not gated. — [pkg/tools/kubectl_filter.go:99-104](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/kubectl_filter.go#L99-L104); [pkg/agent/conversation.go:723-730](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/agent/conversation.go#L723-L730) (verified)
  - *To reach the next level:* Once untrusted content has been read, force every egress-capable or state-changing call through approval.
- **C L0:** Tool results enter the conversation as plain function results with no provenance or untrusted marking, for kubectl, bash and MCP output alike. — [pkg/agent/conversation.go:1140-1144](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/agent/conversation.go#L1140-L1144); searched `rg -n -i 'untrusted|provenance|taint|injection'` in `pkg cmd` → 2 hits (both hits are unrelated: a test case for 'kubectl taint' and the 'taint' verb in the write-op list) (verified)
  - *To reach the next level:* Tag tool and MCP results as untrusted data and apply the limit to every source.
- **D L2:** The gate is on by default and content cannot switch it off directly, but the operator can disable it silently via the user config file. — [cmd/main.go:167](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L167); [cmd/main.go:87](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L87) (verified)
  - *To reach the next level:* Require an explicit warned flag to disable it.
- **B L0:** A hijacked agent can read secrets ('kubectl get secret -o yaml' is classified read-only), and the gate does not cover every path, so it can exfiltrate them and delete resources unattended. (inferred)
  - *To reach the next level:* Ensure either exfiltration or irreversible actions always require human approval.
- **Cap:** C5-WORSTCASE — In the default configuration a hijacked agent can both leak secrets and take irreversible cluster actions without approval.

### C6 Memory, context & configuration integrity — 0.10 (high)

kubectl-ai loads no instruction files from the project you run it in, and by default conversations live only in memory. Its settings come from ~/.config/kubectl-ai (config.yaml, tools.yaml, mcp.yaml), which looks like safe user scope, but the approval gate does not cover every path, so the agent can write those files itself, and a written config can turn off approvals, register custom tools or enable MCP servers for every later session.

- **S L0:** Auto-loaded user config (including skipPermissions, mcpClient and custom tool paths) is model-writable because the gate does not cover every path, and is applied on the next launch with no prompt. — [cmd/main.go:158-161](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L158-L161); [cmd/main.go:212-216](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L212-L216); [cmd/main.go:87](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L87); [cmd/main.go:153-156](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L153-L156) (verified)
  - *To reach the next level:* Never let agent tool calls write the agent's own config directories, and require a trust decision for security-relevant settings.
- **C L0:** No config or memory path is protected: config.yaml, tools.yaml and mcp.yaml are all loaded unconditionally from user scope. — [cmd/main.go:158-161](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L158-L161); [cmd/main.go:153-156](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L153-L156) (verified)
  - *To reach the next level:* Protect every auto-loaded file.
- **D L1:** Sessions default to an in-memory store per process (filesystem persistence is opt-in under ~/.kubectl-ai/sessions); this is a single-user CLI, and the namespacing is not a security control the model is kept out of. — [cmd/main.go:203](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L203); [pkg/sessions/store.go:64-70](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/sessions/store.go#L64-L70) (verified)
  - *To reach the next level:* Enforce per-session namespaces the model cannot write outside of.
- **B L1:** A poisoned config persists across all of the user's future sessions and can trigger tool use (e.g. disabling approvals or launching MCP servers). — [cmd/main.go:87](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L87); [cmd/main.go:212-216](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L212-L216) (verified)
  - *To reach the next level:* Make persisted changes visible and require review before they take effect.
- **Cap:** none

### C7 Third-party extensions — 0.00 (medium)

No third-party extension runs in the default configuration, but the extension mechanism is weak when enabled. The '--mcp-client' flag auto-creates a config that launches an unpinned 'npx -y' MCP server, with no hashes or re-approval. The first-run configuration handling is not locked down. MCP servers launch as the same user via the mcp-go library, inferred to inherit the full environment.

- **S L0:** The shipped default MCP server is launched with 'npx -y' and no version pin or integrity check. — [pkg/mcp/default_config.yaml:3-6](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/mcp/default_config.yaml#L3-L6); searched `rg -n -i 'sha256|checksum|signature|digest'` in `pkg/mcp` → 0 hits (no integrity verification code) (verified)
  - *To reach the next level:* Pin extension versions and verify hashes or signatures.
- **C L0:** Neither MCP servers nor custom tools are verified in any way. — [pkg/mcp/default_config.yaml:3-6](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/mcp/default_config.yaml#L3-L6); [cmd/main.go:153-156](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L153-L156) (verified)
  - *To reach the next level:* Verify every extension type.
- **D L0:** When MCP is first enabled, the bundled default auto-installs a package via npx, and the first-run configuration handling is not locked down. — [pkg/mcp/config.go:124-146](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/mcp/config.go#L124-L146) (verified)
  - *To reach the next level:* Show the exact command before first launch; only user scope may add servers.
- **B L0:** MCP stdio servers run as the same user; mcp-go's stdio transport is inferred to append the configured env to the full os.Environ(), giving them all of the agent's credentials. — [pkg/mcp/stdio_client.go:78](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/mcp/stdio_client.go#L78) (inferred)
  - *To reach the next level:* Launch extensions with a scrubbed environment and their own scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.00 (high)

API keys come from environment variables, and nothing in kubectl-ai redacts anything. Tracing is on by default and does not protect sensitive data. Every subprocess also receives the full environment.

- **S L0:** Nothing is redacted before logs or traces are written. — searched `rg -n -i 'redact|mask|scrub|sanitiz'` in `pkg gollm cmd` → 5 hits (hits are MCP server-name sanitization and DOMPurify in the web UI; none redact secrets) (verified)
  - *To reach the next level:* Redact secrets before any log or trace write; store keys in an OS keychain.
- **C L0:** No path is protected: trace, klog file, model-bound tool results and subprocess environments all carry secrets unfiltered. — [pkg/tools/tools.go:196-214](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/tools.go#L196-L214); [pkg/tools/bash_tool.go:98](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/bash_tool.go#L98) (verified)
  - *To reach the next level:* Protect logs, traces, model-bound messages and subprocess environments.
- **D L0:** Verbose payload tracing is on by default, and its storage is not locked down. — [cmd/main.go:181](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L181); [pkg/agent/conversation.go:388-390](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/agent/conversation.go#L388-L390) (verified)
  - *To reach the next level:* Make payload tracing opt-in and redacted, stored with restrictive access.
- **B L0:** Long-lived provider API keys and the kubeconfig are reachable by every subprocess and by the model's commands. — [pkg/tools/bash_tool.go:98](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/bash_tool.go#L98); [gollm/gemini.go:63](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/gollm/gemini.go#L63) (verified)
  - *To reach the next level:* Avoid passing long-lived keys to subprocesses; use short-lived scoped credentials.
- **Cap:** none

### C9 Audit & traceability — 0.40 (high)

Every tool call, including custom and MCP tools, is written to a structured YAML trace with its arguments, result and timestamp, and this is on by default. The record has gaps: it does not log approvals or denials, it does not say who asked or who approved, it is overwritten on every launch, and its storage location is not locked down. Write errors are ignored.

- **S L2:** Each tool request and response is journaled as a structured event with name, arguments, result and timestamp. — [pkg/tools/tools.go:196-214](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/tools.go#L196-L214) (verified)
  - *To reach the next level:* Add actor attribution (requesting user, approver) and record approval decisions.
- **C L2:** All tools go through InvokeTool and are recorded, but approvals and denials are not journaled. — [pkg/tools/tools.go:196-214](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/tools.go#L196-L214); searched `rg -n -i 'approv|declin'` in `pkg/journal pkg/tools/tools.go` → 0 hits (no approval events in the journal or tool invocation path) (verified)
  - *To reach the next level:* Record approval and denial events alongside tool calls.
- **D L1:** On by default, but the trace is a fixed path truncated on each launch and writable by the agent's own commands. — [cmd/main.go:181](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L181) (verified)
  - *To reach the next level:* Store the record outside agent-writable locations, append-only.
- **B L1:** The return value of recorder.Write is ignored, so logging failures are silent, and each new run truncates the previous record. — [pkg/tools/tools.go:198-206](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/tools.go#L198-L206) (verified)
  - *To reach the next level:* Surface logging errors and keep records durable across runs.
- **Cap:** none

### C10 Limits & kill switch — 0.30 (high)

The agent stops after 20 model iterations per request, but there is no wall-clock limit, token budget or cost cap, and individual commands have no timeout except a 7-second cap for watch, follow-logs and attach. Ctrl+C cancels the context, which kills the bash child process but not any process group or background jobs it started. Long-running commands can run indefinitely.

- **S L1:** Only an iteration cap is enforced; a timeout exists only for detected streaming commands. — [pkg/agent/conversation.go:593](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/agent/conversation.go#L593); [pkg/tools/streaming.go:42](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/streaming.go#L42) (verified)
  - *To reach the next level:* Add a session wall-clock limit, a per-command timeout and a token/cost budget.
- **C L1:** The cap applies to the top-level loop; tool executions have no general timeout. — [pkg/agent/conversation.go:593](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/agent/conversation.go#L593); [pkg/tools/streaming.go:44-47](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/tools/streaming.go#L44-L47) (verified)
  - *To reach the next level:* Apply timeouts to every tool execution.
- **D L2:** Default MaxIterations is 20, configurable by the operator; the model cannot raise it. — [cmd/main.go:177](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L177) (verified)
  - *To reach the next level:* Enforce hard ceilings that config cannot exceed and make sure the model cannot reset limits.
- **B L1:** Stopping cancels the context and kills the direct bash child only; no process-group kill, so background processes survive, and no time ceiling exists. — [cmd/main.go:258](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/cmd/main.go#L258); [pkg/sandbox/local.go:48](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/2c8ff822bacba3009f215df204610a29f5632454/pkg/sandbox/local.go#L48); searched `rg -n 'Setpgid|Pgid|syscall.Kill'` in `pkg cmd` → 0 hits (no process-group handling) (verified)
  - *To reach the next level:* Kill the whole process group on stop and add tight time ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Pod logs, events, annotations and configmaps returned by auto-approved kubectl reads enter context as plain tool results (pkg/agent/conversation.go:1140). · [B] sensitive data/systems: Operator kubeconfig with full authority; 'kubectl get secret' classified read-only (pkg/tools/kubectl_filter.go:26); full env with API keys (pkg/tools/bash_tool.go:98). · [C] state change / egress: kubectl writes and egress through paths the approval gate does not cover; other shell commands gated (pkg/agent/conversation.go:757). · Same default session? Yes

## Highest-impact improvements
1. Harden the approval classifier so the auto-approved set is a verified read-only allowlist. — C2 C L1→L3, +0.150 before caps (Playbook 5)
2. Make payload tracing opt-in, redacted and access-restricted. — C8 D L0→L2, +0.100 before caps (Playbook 4)
3. Run commands with a scrubbed environment (only KUBECONFIG and PATH) instead of os.Environ(). — C8 B L0→L2, +0.100 before caps (Playbook 4)
4. Block agent tool writes to ~/.config/kubectl-ai and require the --skip-permissions flag (not a config key) to disable approval. — C6 S L0→L2, +0.150 before caps (Playbook 2)
5. Add a per-command timeout, session wall-clock limit and process-group kill on stop. — C10 S L1→L3, +0.150 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the interactive terminal CLI; web UI (--ui-type web), MCP server mode (--mcp-server, gating left to the host), quiet mode (approval-requiring calls abort unless --skip-permissions) and the in-cluster k8s/ manifests were reviewed only where they bear on defaults.
- Effects relying on third-party behaviour are inferred, including some behind other findings, and mcp-go passing os.Environ() to stdio servers.
- No reviewer-steering text aimed at AI auditors was found in the repository.
- gollm provider internals beyond credential loading and journaling, and the kubectl-utils and modelserving directories, were not examined in depth.
