# Defense-in-Depth Score: kagent

**Repo:** https://github.com/kagent-dev/kagent · **Commit:** `72a28f01403ab48f7d6df02d67e24e937c3a0459` · **Reviewed:** 2026-10-03
**What it is:** Kubernetes-native framework/platform for running AI agents declaratively in clusters, with built-in k8s/Helm/Istio/Prometheus tools
**Category:** Infrastructure & Ops
**Scored configuration:** Helm chart defaults (controller.auth.mode=insecure, default NoopAuthorizer, cluster-scoped RBAC, kagent-tools, grafana-mcp and kmcp enabled, bundled Postgres) with the required Agent Substrate runtime on its default gVisor WorkerPool class, agents on the default kagent Go ADK harness.
**Agent surface (default):** code execution opt-in · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 3.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L0 | L0 | 0.15 | G2 | **0.15** | High |
| C2 | Approval gates | L3 | L1 | L0 | L1 | 0.35 | C2-SELFAPPROVE | **0.25** | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | Medium |
| C4 | Code-execution isolation | L4 | L3 | L3 | L1 | 0.72 | — | **0.72** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L1 | L2 | 0.30 | — | **0.30** | High |
| C7 | Third-party extensions | L3 | L2 | L2 | L1 | 0.53 | — | **0.53** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | Medium |
| C9 | Audit & traceability | L2 | L2 | L1 | L2 | 0.45 | — | **0.45** | Medium |
| C10 | Limits & kill switch | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |


kagent runs each agent in a gVisor-isolated Substrate actor whose network egress is limited to configured origins and whose model and MCP credentials are injected by an egress gateway rather than placed in the agent's environment - a genuinely strong isolation design. That design is undercut by the control plane it ships with: the default controller authentication mode is 'insecure' (any caller can claim any user, defaulting to admin) with a permit-all authorizer, and every agent actor is always allowed to reach that controller API, which can create agents, tool servers and model configs and call any registered MCP tool under a cluster-wide service account. Human approval of tool calls exists but is off by default, and there are no step, time or spend limits on the agent loop.

## Critical gaps
- Default controller authentication accepts any caller-asserted user (default admin@kagent.dev) with a permit-all authorizer, and every agent actor is always allowed to reach that controller. (ASI03, T3, T9; C1) — [helm/kagent/values.yaml:227-230](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/helm/kagent/values.yaml#L227-L230); [go/core/internal/httpserver/auth/authn.go:24-31](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/httpserver/auth/authn.go#L24-L31); [go/core/pkg/app/app.go:128-130](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/pkg/app/app.go#L128-L130); [go/core/internal/translator/kagent/compiler.go:71](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/translator/kagent/compiler.go#L71)
- The controller ServiceAccount holds cluster-wide create/update/patch/delete on all core, apps and batch resources, reachable through the unauthenticated API. (ASI03, T3; C1) — [helm/kagent/templates/rbac/writer-role.yaml:50-58](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/helm/kagent/templates/rbac/writer-role.yaml#L50-L58)
- Under the default insecure authentication any caller can assert the session owner's identity and answer a pending tool-approval request, so approval can come from a non-principal. (ASI09, T10, T9; C2) — [go/core/internal/httpserver/auth/authn.go:24-31](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/httpserver/auth/authn.go#L24-L31); [go/adk/pkg/a2a/hitl.go:389](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/adk/pkg/a2a/hitl.go#L389)

## Criterion details

### C1 Identity & least privilege — 0.15 (high)

Agent actors carry no Kubernetes service-account credentials of their own, and model/MCP keys are injected outside the actor, which is a good start. But every agent can always reach the kagent controller API, and by default that API trusts whatever user the caller claims (falling back to admin) and authorizes every action. Through it a caller can create or change agents, tool servers, model configs and call any registered MCP tool, all executed with the controller's cluster-wide service account that can create, update and delete every core, apps and batch resource. Least privilege therefore depends entirely on the operator switching to trusted-proxy mode and supplying a real authorizer.

- **S L1:** Actors have a dedicated runtime identity, but the controller API they can always reach acts with a cluster-wide service account and accepts any caller-asserted user identity with a permit-all authorizer. — [go/core/internal/httpserver/auth/authn.go:24-31](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/httpserver/auth/authn.go#L24-L31); [go/core/pkg/app/app.go:128-130](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/pkg/app/app.go#L128-L130); [helm/kagent/templates/rbac/writer-role.yaml:50-58](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/helm/kagent/templates/rbac/writer-role.yaml#L50-L58); [go/core/internal/translator/kagent/compiler.go:71](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/translator/kagent/compiler.go#L71) (verified)
  - *To reach the next level:* No per-tool or per-capability scoping: one controller ServiceAccount with wildcard core/apps/batch write serves every request.
- **C L1:** Owner checks exist in the session/sandbox services, but other paths (CallMCPAppTool, agent/template/tool-server CRUD) only consult the authorizer, which is NoopAuthorizer by default. — [go/core/internal/service/tool/service.go:262-277](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/service/tool/service.go#L262-L277); [go/core/internal/grpcserver/policy.go:46](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/grpcserver/policy.go#L46); [go/core/pkg/app/app.go:128-130](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/pkg/app/app.go#L128-L130) (verified)
  - *To reach the next level:* Not every tool path goes through an authorization layer that actually restricts anything by default.
- **D L0:** The shipped chart sets controller auth mode to insecure and cluster-scoped RBAC; least privilege requires manual hardening. — [helm/kagent/values.yaml:227-230](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/helm/kagent/values.yaml#L227-L230); [helm/kagent/values.yaml:193-206](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/helm/kagent/values.yaml#L193-L206) (verified)
  - *To reach the next level:* Default should require authenticated principals and a restrictive authorizer, with namespace-scoped RBAC.
- **B L0:** If the authorization layer fails (as it does by default), the controller ServiceAccount can create/update/delete all core resources cluster-wide, including Secrets and Pods. — [helm/kagent/templates/rbac/writer-role.yaml:50-58](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/helm/kagent/templates/rbac/writer-role.yaml#L50-L58); [helm/kagent/templates/rbac/getter-role.yaml:60-66](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/helm/kagent/templates/rbac/getter-role.yaml#L60-L66) (verified)
  - *To reach the next level:* Blast radius should be limited to one namespace/tenant and mostly read-only.
- **Cap:** G2 — The default InsecureAuthenticator takes the acting user from the caller-controlled X-User-Id header or user_id query parameter, so any client - including an agent actor, whose egress always allows the controller - can impersonate any user at runtime.

### C2 Approval gates — 0.25 (high)

kagent has a real human-in-the-loop mechanism: an MCP tool binding can set requireApproval, which pauses the task and shows the approver the exact tool name and arguments before resuming the same stored call. It is off by default, though, and it only covers MCP tool bindings - the skills bash tool, the Claude harness (run with --dangerously-skip-permissions) and the Codex harness (danger-full-access) are not gated, and the controller's CallMCPAppTool API can invoke any registered MCP tool directly without any approval. Kubernetes writes made through these tools are generally irreversible.

- **S L3:** When enabled, each call of a bound MCP tool pauses as an A2A input-required task carrying the exact tool name and arguments, and the decision resumes that stored call. — [go/api/v1alpha3/agenttemplate_types.go:61-65](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/api/v1alpha3/agenttemplate_types.go#L61-L65); [go/adk/pkg/mcp/registry.go:193-195](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/adk/pkg/mcp/registry.go#L193-L195); [go/adk/pkg/a2a/hitl.go:389](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/adk/pkg/a2a/hitl.go#L389) (verified)
  - *To reach the next level:* No argument-level allow/deny/escalate policy; approval is all-or-nothing per binding.
- **C L1:** Only MCP bindings with requireApproval are gated; bash (skills), the Claude/Codex native tools and the controller CallMCPAppTool path bypass it. — [go/adk/pkg/tools/skills.go:281-284](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/adk/pkg/tools/skills.go#L281-L284); [go/harness/claude/internal/driver/process.go:129](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/harness/claude/internal/driver/process.go#L129); [go/harness/codex/internal/driver/process.go:137](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/harness/codex/internal/driver/process.go#L137); [go/core/internal/service/tool/service.go:262-277](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/service/tool/service.go#L262-L277) (verified)
  - *To reach the next level:* Every tool path, including built-in shell, harness-native tools and the controller's direct MCP tool call API, should traverse the gate.
- **D L0:** requireApproval defaults to false (omitempty bool) and no shipped template sets it. — [go/api/v1alpha3/agenttemplate_types.go:61-65](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/api/v1alpha3/agenttemplate_types.go#L61-L65) (verified)
  - *To reach the next level:* Approval should be on by default for mutating tools.
- **B L1:** Bound Kubernetes/Helm tools can make destructive cluster changes that kagent cannot undo; checkpoints cover only agent conversation state. — [helm/kagent/templates/toolserver-kagent.yaml:21-23](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/helm/kagent/templates/toolserver-kagent.yaml#L21-L23) (verified)
  - *To reach the next level:* No rollback or dry-run for external actions; no rate limits on consequential actions.
- **Cap:** C2-SELFAPPROVE — Approval decisions arrive as A2A messages authenticated by the default InsecureAuthenticator, so any caller that sets X-User-Id to the session owner (including another agent actor, which can always reach the controller) can approve a pending call.

### C3 Tool & action scoping — 0.45 (medium)

Tool bindings can restrict an agent to a named subset of an MCP server's tools, and the built-in file tools resolve symlinks and require the result to stay inside the session or skills directory. The bash tool registered with skills is raw shell passthrough, and argument validation for the bundled Kubernetes tools lives in the separate kagent-tools server, which is not in this repository. An omitted tool list exposes every tool on the server.

- **S L2:** File tools use EvalSymlinks plus root containment, but bash takes an arbitrary shell string and MCP tool arguments get no framework-side validation. — [go/adk/pkg/tools/skills.go:370-379](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/adk/pkg/tools/skills.go#L370-L379); [go/adk/pkg/tools/shell.go:273](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/adk/pkg/tools/shell.go#L273) (verified)
  - *To reach the next level:* No allowlist validation for shell or MCP arguments; general tools are not replaced by narrow ones.
- **C L2:** Built-in file tools validate paths; MCP extension tools and bash have no shared validation layer. — [go/api/v1alpha3/agenttemplate_types.go:53-60](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/api/v1alpha3/agenttemplate_types.go#L53-L60) (verified)
  - *To reach the next level:* Extension tools should be wrapped by a shared validation layer.
- **D L2:** Tool subsets are selectable per binding, but an omitted list exposes every server tool and the default tool server includes write tools. — [go/api/v1alpha3/agenttemplate_types.go:53-60](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/api/v1alpha3/agenttemplate_types.go#L53-L60) (verified)
  - *To reach the next level:* Default should be a read-only tool set with write/exec requiring explicit enabling.
- **B L1:** Misused Kubernetes tools reach the cluster with the tool server's authority; bash is confined to the actor. — [helm/kagent/templates/toolserver-kagent.yaml:21-23](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/helm/kagent/templates/toolserver-kagent.yaml#L21-L23) (inferred)
  - *To reach the next level:* Tools should be scoped and quantity-bounded.
- **Cap:** none

### C4 Code-execution isolation — 0.72 (high)

Every agent runs as a Substrate actor whose sandbox class defaults to gVisor (microVM optional), there is no host-execution fallback, standalone sandboxes run the same way with no egress, and the actor's network egress is compiled into a DNS-origin allowlist. The concern is what sits inside that allowlist: the controller API endpoint is always added, and in the default insecure mode it lets code in the actor create agents and tool servers (kmcp MCPServer pods run outside Substrate) and call any MCP tool. Actors also keep a durable /data directory across restarts.

- **S L4:** Actor templates select the gVisor sandbox class by default and MicroVM optionally; unsupported classes are rejected. — [go/core/internal/substrate/actor_template.go:217-229](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/substrate/actor_template.go#L217-L229) (verified)
- **C L3:** All harness execution (Go/Python ADK, Claude, Codex, BYO) and standalone sandboxes run as Substrate actors; the controller API can still create kmcp MCPServer pods that run outside Substrate. — [go/core/internal/substrate/actor_template.go:217-229](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/substrate/actor_template.go#L217-L229); [go/core/internal/service/tool/service.go:161-166](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/service/tool/service.go#L161-L166) (verified)
  - *To reach the next level:* Processes created via the control plane (kmcp MCPServer deployments) are not confined to the sandbox.
- **D L3:** The sandbox class comes from the operator's WorkerPool and there is no unsandboxed mode; but the egress allowlist is derived from Agent configuration that the unauthenticated API can modify for future sessions. — [go/core/internal/translator/revision.go:85-87](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/translator/revision.go#L85-L87); [go/core/internal/grpcserver/policy.go:66](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/grpcserver/policy.go#L66) (verified)
  - *To reach the next level:* Sandbox and egress policy should be defined outside anything reachable from the actor.
- **B L1:** Egress is allowlisted and credentials are not in the actor env, but the allowlist always includes the controller API, which by default grants admin-equivalent control-plane access. — [go/core/internal/translator/kagent/compiler.go:71](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/translator/kagent/compiler.go#L71); [go/core/internal/substrate/policy.go:19-34](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/substrate/policy.go#L19-L34); [go/core/internal/httpserver/auth/authn.go:24-31](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/httpserver/auth/authn.go#L24-L31) (verified)
  - *To reach the next level:* The allowlist should not include an unauthenticated control-plane endpoint with cluster-wide authority.
- **Cap:** none

### C5 Untrusted input blast radius — 0.05 (high)

Nothing in kagent tracks whether untrusted content (Kubernetes logs and objects, MCP tool results, sub-agent replies, A2A messages from other callers) has entered a session, and approval is not tied to it. The main structural limit is Substrate's egress allowlist, which prevents a hijacked agent from posting data to arbitrary internet hosts. A hijacked agent with Kubernetes tools bound can still make irreversible cluster changes without a human by default.

- **S L0:** No taint tracking, provenance marking, or approval tied to untrusted content in the agent runtime. — searched `rg -n -i 'untrusted|taint|provenance|quarantin|spotlight'` in `go/adk/pkg` → 1 hits (The single hit is a test fixture header value in registry_test.go, not a control.) (verified)
  - *To reach the next level:* Disable or gate egress/state-changing tools once untrusted content is read.
- **C L0:** Tool results enter context with the same standing as user instructions. — [go/adk/pkg/mcp/registry.go:139](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/adk/pkg/mcp/registry.go#L139) (verified)
  - *To reach the next level:* Untrusted sources should be distinguished at all.
- **D L0:** No control exists to be on by default. — searched `rg -n -i 'untrusted|taint|provenance|quarantin|spotlight'` in `go/adk/pkg` → 1 hits (Test fixture only.) (verified)
  - *To reach the next level:* An untrusted-content control on by default.
- **B L1:** Irreversible cluster actions via bound tools are unattended by default, while exfiltration to arbitrary hosts is blocked by the actor egress allowlist. — [go/core/internal/substrate/policy.go:19-34](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/substrate/policy.go#L19-L34); [go/api/v1alpha3/agenttemplate_types.go:61-65](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/api/v1alpha3/agenttemplate_types.go#L61-L65) (verified)
  - *To reach the next level:* Irreversible actions should require human approval after untrusted input.
- **Cap:** none
- **Notes:** If a shell is present (skills, Claude or Codex harness), the always-allowed controller endpoint lets a hijacked agent read other users' sessions under the default insecure auth; that configuration would rate B one level lower.

### C6 Memory, context & configuration integrity — 0.30 (high)

Long-term memory is opt-in (pgvector disabled by default); when enabled, the model's save_memory tool writes arbitrary text that is preloaded into later sessions, and memory scoping is not enforced on every path. There are no auto-loaded workspace instruction files; skills are pinned to immutable commits. Agent system prompts and tool bindings live in AgentTemplates that the default unauthenticated API can rewrite, which would persist into every future session.

- **S L1:** save_memory stores any model-provided content without validation and preload_memory re-injects it; agent configuration is writable through the unauthenticated API. — [go/adk/pkg/memory/save_memory_tool.go:16-41](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/adk/pkg/memory/save_memory_tool.go#L16-L41); [go/adk/pkg/agent/agent.go:215-219](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/adk/pkg/agent/agent.go#L215-L219) (verified)
  - *To reach the next level:* Memory writes should be gated or validated with expiry, and configuration changes should need an explicit trust decision.
- **C L1:** Prompt and skill sources are operator-scoped and skills are commit-pinned, but memory and template rewrites are uncontrolled. — [go/core/internal/skillsinit/git.go:53-56](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/skillsinit/git.go#L53-L56); [go/core/internal/grpcserver/policy.go:66](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/grpcserver/policy.go#L66) (verified)
  - *To reach the next level:* All memory stores and configuration paths should be controlled.
- **D L1:** Memory queries filter by agent and user, but that scoping is not enforced on every path. (verified)
  - *To reach the next level:* Namespaces should be enforced from the authenticated identity.
- **B L2:** By default persistence is session history; with memory enabled, poisoned entries persist across a user's sessions and can steer tool use. — [helm/kagent/values.yaml:113-116](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/helm/kagent/values.yaml#L113-L116) (verified)
  - *To reach the next level:* Persistent state should be session-scoped or easily inspected and purged.
- **Cap:** none

### C7 Third-party extensions — 0.53 (high)

Code-bearing extensions are pinned: Harness images must be referenced by sha256 digest and skills by a full git commit or S3 object version, and the v1alpha3 compiler emits only HTTP/SSE MCP servers, not locally launched stdio servers. Remote MCP servers are not pinned, so their tool definitions can change between sessions without re-approval. Skill scripts run inside the agent's own actor with its full environment (which holds only credential placeholders).

- **S L3:** Images require a sha256 digest and skills a full commit SHA, giving integrity on code extensions. — [go/api/v1alpha3/harness_types.go:115-116](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/api/v1alpha3/harness_types.go#L115-L116); [go/core/internal/skillsinit/git.go:53-56](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/skillsinit/git.go#L53-L56) (verified)
  - *To reach the next level:* No re-approval when an MCP server's tool definitions change.
- **C L2:** Images, skills and plugins are pinned; remote MCP servers are not verified. — [go/adk/pkg/mcp/registry.go:101-118](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/adk/pkg/mcp/registry.go#L101-L118) (verified)
  - *To reach the next level:* MCP servers should also be pinned/verified.
- **D L2:** Extensions are added via Kubernetes resources, but the default unauthenticated API can also add tool servers (including kmcp MCPServer deployments). — [go/core/internal/service/tool/service.go:161-166](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/service/tool/service.go#L161-L166); [go/core/internal/grpcserver/policy.go:42](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/grpcserver/policy.go#L42) (verified)
  - *To reach the next level:* Only authenticated admin scope should be able to add extensions, showing what will run.
- **B L1:** Skill code runs via bash as a subprocess in the same actor with the agent's full environment and controller access. — [go/adk/pkg/tools/shell.go:273-274](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/adk/pkg/tools/shell.go#L273-L274) (verified)
  - *To reach the next level:* Extensions should run with a scrubbed environment and their own scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.50 (medium)

Model API keys and Secret-backed MCP headers never enter the agent: the compiler replaces them with an inert placeholder and the Substrate egress gateway injects the real header only for the bound destination. Tool logging records only tool names and result keys, and OpenTelemetry export and message-content capture are off by default. There is no redaction of secrets returned by tools (for example Kubernetes Secret reads) before they reach the model or the stored history, the bundled Postgres ships fixed credentials, and credential binding through the control-plane API is not locked down.

- **S L2:** Gateway credential injection with in-actor placeholders keeps keys out of the runtime, but there is no redaction of secrets in tool output, history or logs. — [go/core/internal/translator/credentials.go:15-20](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/translator/credentials.go#L15-L20); [go/core/internal/substrate/policy.go:64](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/substrate/policy.go#L64) (verified)
  - *To reach the next level:* Redaction before logs and model-bound messages on all major paths.
- **C L2:** Subprocess env, model-bound prompts, logs and telemetry are free of provider keys by design, but secrets returned by tools reach model-bound messages and stored A2A history unredacted. — [go/adk/pkg/agent/agent.go:513-516](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/adk/pkg/agent/agent.go#L513-L516); [helm/kagent/values.yaml:1008-1011](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/helm/kagent/values.yaml#L1008-L1011) (verified)
  - *To reach the next level:* Tool outputs sent to the model and stored A2A history should also be covered.
- **D L2:** Telemetry and content capture are opt-in, but the bundled database uses hardcoded credentials and the tool server logs at debug level by default. — [helm/kagent/values.yaml:150-151](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/helm/kagent/values.yaml#L150-L151); [helm/kagent/values.yaml:839-840](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/helm/kagent/values.yaml#L839-L840) (verified)
  - *To reach the next level:* No plaintext default credentials; redaction always on.
- **B L2:** Keys are long-lived Kubernetes Secrets, and credential binding through the control-plane API is not locked down. (inferred)
  - *To reach the next level:* Credential bindings should be authorized and keys short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.45 (medium)

Every task's messages and events are persisted by the controller into PostgreSQL through a TaskStore API, and the runtime stops if a save fails, so there is a durable conversation and tool-call history outside the agent. Runtime logs record each tool start and completion by name and call ID, though not arguments. Attribution is weak by default: the user identity is whatever the caller claims, and the runtime's TaskStore identity is an unsigned header the project itself labels insecure.

- **S L2:** A2A history including function-call parts is persisted per task in PostgreSQL; tool callbacks log tool name, call ID and session. — [go/adk/pkg/agent/agent.go:487-494](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/adk/pkg/agent/agent.go#L487-L494); [go/adk/pkg/a2a/converter.go:61-65](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/adk/pkg/a2a/converter.go#L61-L65) (inferred)
  - *To reach the next level:* Actor attribution cannot be trusted by default (caller-asserted user, unsigned runtime identity).
- **C L2:** All agent tool calls flow through the ADK event stream; calls made directly via the controller's CallMCPAppTool are not part of any agent task record. — [go/core/internal/service/tool/service.go:262-277](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/service/tool/service.go#L262-L277) (verified)
  - *To reach the next level:* Direct controller-side tool calls and configuration changes should be recorded too.
- **D L1:** Records are written by the controller, but the runtime identity is an unsigned header that a process in an actor could forge. — [go/core/internal/service/taskstore/authority.go:15-23](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/core/internal/service/taskstore/authority.go#L15-L23) (verified)
  - *To reach the next level:* Records should be written by a component the agent cannot impersonate.
- **B L2:** Task persistence failures are surfaced and stop the run. — [go/adk/pkg/taskstore/executor.go:124-128](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/adk/pkg/taskstore/executor.go#L124-L128) (verified)
  - *To reach the next level:* Durable per-action records with replayable trajectories.
- **Cap:** none

### C10 Limits & kill switch — 0.15 (high)

The agent loop has no step, wall-clock or spend limit: no iteration cap exists in the Go ADK runtime and the controller's A2A client timeout defaults to none. Individual tools have timeouts (30-60 seconds for bash, 30 seconds for MCP servers) and tasks can be cancelled through A2A. Session idle expiry never applies to running tasks, so a runaway task is bounded only by the operator noticing and cancelling it.

- **S L1:** Only per-tool timeouts and A2A task cancellation exist; no iteration, wall-clock or cost cap. — searched `rg -n -i 'max_?llm_?calls|max_?iterations|maxturns|max_?steps'` in `go/adk/pkg go/core/internal/translator/adkconfig` → 0 hits (No loop limit in the runtime or the compiled agent config.); [go/adk/pkg/tools/shell.go:265-271](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/go/adk/pkg/tools/shell.go#L265-L271) (verified)
  - *To reach the next level:* An iteration cap plus a wall-clock or token/cost cap enforced in code.
- **C L1:** Timeouts apply to individual bash and MCP calls only. — [helm/kagent/templates/toolserver-kagent.yaml:21](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/helm/kagent/templates/toolserver-kagent.yaml#L21) (verified)
  - *To reach the next level:* Limits should cover the top-level loop and sub-agents.
- **D L0:** The loop is unlimited by default and the A2A client timeout defaults to none. — [helm/kagent/values.yaml:245-253](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/helm/kagent/values.yaml#L245-L253) (verified)
  - *To reach the next level:* Sensible default step/time/cost ceilings.
- **B L0:** A runaway task can loop and spend indefinitely; idle expiry excludes running tasks. — [helm/kagent/values.yaml:416-418](https://github.com/kagent-dev/kagent/blob/72a28f01403ab48f7d6df02d67e24e937c3a0459/helm/kagent/values.yaml#L416-L418) (verified)
  - *To reach the next level:* Tight per-run time and cost ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: MCP tool results (Kubernetes logs, resources, Grafana data) and A2A messages from any caller enter model context unmarked (go/adk/pkg/mcp/registry.go:139; go/core/internal/httpserver/auth/authn.go:24-31) · [B] sensitive data/systems: Cluster state via kagent-tools and the controller API, other users' sessions and memory via the unauthenticated control plane (helm/kagent/templates/rbac/getter-role.yaml:60-66) · [C] state change / egress: Bound MCP tools change cluster state without approval by default (go/api/v1alpha3/agenttemplate_types.go:65); controller API always reachable from actors (go/core/internal/translator/kagent/compiler.go:71) · Same default session? Yes

## Highest-impact improvements
1. Ship controller.auth.mode=trusted-proxy (or verified JWT) with a deny-by-default authorizer, and stop honoring X-User-Id from unauthenticated callers. — C1 D L0→L2, +0.100 before caps (Playbook 4)
2. Expose only the TaskStore runtime methods on the address added to every actor's egress allowlist (separate listener/port), so agents cannot reach the admin API. — C4 B L1→L3, +0.100 before caps (Playbook 3)
3. Default requireApproval to true for MCP bindings to write-capable tool servers and gate CallMCPAppTool, bash and harness-native shells through the same HITL path. — C2 D L0→L2, +0.100 before caps (Playbook 5)
4. Add a per-task iteration cap and wall-clock deadline in the Go ADK executor with sensible defaults. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
5. Bind memory namespaces to the authenticated principal. — C6 D L1→L2, +0.050 before caps (Playbook 2)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The bundled kagent-tools subchart (v0.3.0, kagent-dev/tools) and kmcp, Substrate, oauth2-proxy subcharts live in other repositories; their RBAC, tool implementations and sandbox enforcement were not examined. Ratings that depend on what kagent-tools can do in the cluster are marked inferred.
- Substrate's gVisor/MicroVM enforcement, egress gateway and credential provider are external; this audit verifies that kagent selects and configures them, not how Substrate enforces them.
- Scored the default kagent Go ADK harness; the Claude harness runs Claude Code with --dangerously-skip-permissions and the Codex harness with danger-full-access (both inside actors), and the Python ADK was not reviewed in depth.
- Architecture docs were used as leads only; docs/architecture/human-in-the-loop.md says MCP bindings expose no approval policy, but the code does (requireApproval), and the code was scored.
- No reviewer-injection attempts were found in repository text.
