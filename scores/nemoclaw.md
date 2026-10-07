# Defense-in-Depth Score: NemoClaw

**Repo:** https://github.com/NVIDIA/NemoClaw · **Commit:** `e68bcacd808a916b6b7e07334d1256ce51f9c6ef` (v0.0.130-12-ge68bcacd) · **Reviewed:** 2026-10-04
**What it is:** NVIDIA reference stack for running always-on AI agents (OpenClaw by default) inside OpenShell sandboxes with managed inference, network policy and lifecycle tooling.
**Category:** AI Assistants
**Scored configuration:** Default `nemoclaw onboard` / express install: OpenClaw agent in an OpenShell sandbox, Balanced policy tier, NVIDIA Endpoints inference, no messaging channels or extra presets.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents opt-in · external communication opt-in

## Score: 4.7 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L3 | L3 | L3 | L3 | 0.75 | — | **0.75** | High |
| C2 | Approval gates | L2 | L1 | L2 | L2 | 0.42 | — | **0.42** | High |
| C3 | Tool & action scoping | L2 | L3 | L2 | L2 | 0.57 | — | **0.57** | High |
| C4 | Code-execution isolation | L3 | L3 | L3 | L2 | 0.70 | — | **0.70** | High |
| C5 | Untrusted input blast radius | L2 | L2 | L3 | L1 | 0.50 | — | **0.50** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C7 | Third-party extensions | L0 | L1 | L1 | L1 | 0.17 | C7-RCELOAD | **0.17** | High |
| C8 | Secrets & sensitive-data protection | L3 | L3 | L3 | L2 | 0.70 | — | **0.70** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | Medium |
| C10 | Limits & kill switch | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | High |


NemoClaw puts a strong box around the agent: every command runs in a hardened, non-root OpenShell sandbox, egress is deny-by-default with operator approval for new hosts, and API keys never enter the sandbox. Inside the box it adds little: no approval for commands, no integrity protection for the agent's self-writable config, hooks, memory and cron jobs, and no cost cap. The dominant risk is that the default Balanced tier still leaves unattended outbound channels (POST to clawhub.ai/openclaw.ai, uninspected npm traffic) and lets the agent install arbitrary packages, so a prompt-injected agent can exfiltrate sandbox data and persist itself.

## Critical gaps
- In the default Balanced tier the agent can install and run arbitrary npm/PyPI/Homebrew packages and ClawHub skills at the model's choice with no consent step. (ASI04, T17, LLM03; C7) — [nemoclaw-blueprint/policies/presets/npm.yaml:16-19](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/presets/npm.yaml#L16-L19); [nemoclaw-blueprint/policies/tiers.yaml:23-31](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/tiers.yaml#L23-L31); [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:121-132](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L121-L132)

## Criterion details

### C1 Identity & least privilege — 0.75 (high)

The agent runs inside the sandbox as an unprivileged 'sandbox' user and never receives the raw provider API keys: NemoClaw registers keys with the OpenShell gateway and writes only placeholder references into the agent config, which the egress proxy swaps for the real key only on the matching allowed endpoint. Inference goes through a virtual inference.local route that the host owns. On the host side, the NemoClaw CLI passes child processes an allowlisted environment rather than its full environment. Credentials are long-lived API keys rather than per-task tokens, and the agent can still spend inference budget freely through the managed route.

- **S L3:** Each credential is bound to one provider endpoint and substituted by the egress proxy from a placeholder; the sandbox identity is a non-root user with no raw keys. — [scripts/generate-openclaw-config.mts:1112](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/scripts/generate-openclaw-config.mts#L1112); [src/lib/adapters/openshell/provider-adapter-cli.ts:535](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/src/lib/adapters/openshell/provider-adapter-cli.ts#L535); [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:62](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L62) (verified)
  - *To reach the next level:* Credentials are long-lived provider keys, not short-lived task-scoped tokens issued per request.
- **C L3:** Every sandbox network path goes through the OpenShell proxy that performs credential substitution, and host-side subprocesses get an allowlisted environment. — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:91](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L91); [src/lib/subprocess-env.ts:97-112](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/src/lib/subprocess-env.ts#L97-L112) (verified)
  - *To reach the next level:* Authorization is not evaluated against a requesting principal (no per-user authorization for messaging senders in NemoClaw's own code).
- **D L3:** Default onboarding attaches only the inference credential (plus an optional search key); GitHub, messaging, Jira and mail credentials require explicit presets. — [nemoclaw-blueprint/policies/tiers.yaml:23-31](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/tiers.yaml#L23-L31); [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:105-108](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L105-L108) (verified)
  - *To reach the next level:* Nothing makes elevation time-bounded; approved policy and credential bindings persist until the sandbox is recreated.
- **B L3:** A hijacked agent can use the managed inference route and the sandbox's own files but cannot read the provider keys or reach other systems' credentials. — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:91](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L91); [scripts/generate-openclaw-config.mts:1112](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/scripts/generate-openclaw-config.mts#L1112) (verified)
  - *To reach the next level:* Keys are not short-lived or automatically revoked after use.
- **Cap:** none

### C2 Approval gates — 0.42 (high)

The only human approval NemoClaw sets up is at the network layer: when the agent tries to reach a host that is not in its policy, OpenShell blocks the request and asks the operator to approve or deny it. Commands, file writes and calls to already-allowed endpoints (including POST to clawhub.ai and openclaw.ai) run without any approval that NemoClaw configures; tool approval is delegated to OpenClaw, and NemoClaw writes no exec-approval policy into the agent config. A startup watcher automatically approves OpenClaw device scope-upgrade requests from allowlisted client IDs that the code itself calls spoofable. Approved endpoints are stored as durable policy revisions until the sandbox is rebuilt.

- **S L2:** Egress to an unlisted host waits for per-request operator approval in the OpenShell TUI, but nothing in NemoClaw gates commands or file changes inside the sandbox. — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:1-14](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L1-L14); searched `rg -n "exec-approvals|askFallback|tools\.exec|maxTurns|maxIterations|maxSteps"` in `scripts/generate-openclaw-config.mts` → 0 hits (NemoClaw emits no OpenClaw exec-approval policy, step cap, or iteration cap into the generated agent config.) (verified)
  - *To reach the next level:* No per-call approval showing the exact command or diff for consequential in-sandbox actions.
- **C L1:** Only unlisted egress crosses a gate; shell commands, file writes and POSTs to baseline-allowed hosts do not, and device scope upgrades are auto-approved. — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:121-132](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L121-L132); [scripts/nemoclaw-start.sh:2092-2095](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/scripts/nemoclaw-start.sh#L2092-L2095); [src/lib/actions/sandbox/auto-pair-approval.ts:19-23](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/src/lib/actions/sandbox/auto-pair-approval.ts#L19-L23) (verified)
  - *To reach the next level:* Consequential tool paths (exec, writes, allowed-endpoint POSTs) do not traverse a gate NemoClaw owns.
- **D L2:** The egress gate is on by default, but approved endpoints accumulate as durable policy revisions and operators can widen policy with a plain policy command. — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:5-14](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L5-L14) (verified)
  - *To reach the next level:* Persisted approvals are not time-bounded and widening is not loudly flagged.
- **B L2:** Most in-sandbox damage is recoverable by recreating the sandbox from the blueprint, but outbound POSTs to allowed hosts are irreversible and unbounded. — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:121-132](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L121-L132); [nemoclaw-blueprint/blueprint.yaml:35](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/blueprint.yaml#L35) (verified)
  - *To reach the next level:* No checkpoints/rollback of sandbox state and no previews or quantity limits on external actions.
- **Cap:** none

### C3 Tool & action scoping — 0.57 (high)

NemoClaw does not narrow the agent's tools themselves (the agent keeps a general shell), but it scopes what those tools can reach with a deny-by-default egress policy that pins host, port, HTTP method, path and even the calling binary. The baseline is tight for inference (only specific NVIDIA API paths), but it allows GET and POST to any path on clawhub.ai and openclaw.ai, and the default Balanced tier adds npm registry access with no request inspection at all. Every tool's network traffic goes through this one policy layer. Inside the sandbox, the agent has full write access to its home and config tree.

- **S L2:** Host/method/path/binary allowlists are enforced at the proxy, but the agent's tools are general (shell) and several default entries are wide (POST /** on clawhub.ai, L4 pass-through npm). — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:70-78](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L70-L78); [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:121-132](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L121-L132); [nemoclaw-blueprint/policies/presets/npm.yaml:16-19](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/presets/npm.yaml#L16-L19) (verified)
  - *To reach the next level:* Default allowlist entries are not narrowed to specific paths for every host, and general tools are not replaced by narrow ones.
- **C L3:** Every process in the sandbox, whatever tool spawned it, is subject to the same egress policy layer. — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:91](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L91); [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:105-108](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L105-L108) (verified)
  - *To reach the next level:* Validation is network-only; filesystem and command arguments are not checked by a central policy layer.
- **D L2:** Policy tiers are selectable, but the default Balanced tier enables package registries and search on top of shell and write tools. — [src/lib/onboard/policy-selection-prompts.ts:131](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/src/lib/onboard/policy-selection-prompts.ts#L131); [nemoclaw-blueprint/policies/tiers.yaml:23-31](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/tiers.yaml#L23-L31) (verified)
  - *To reach the next level:* Default is not a read-only tool set.
- **B L2:** A misused tool is confined to the sandbox with full write inside it plus the allowed endpoints. — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:45](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L45); [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:121-132](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L121-L132) (verified)
  - *To reach the next level:* No quantity bounds on writes or outbound requests.
- **Cap:** none

### C4 Code-execution isolation — 0.70 (high)

The whole agent, including every command it runs, lives inside an OpenShell container sandbox: non-root sandbox user, read-only system directories, Landlock filesystem rules, seccomp and no-new-privileges applied by OpenShell, and a separate network namespace whose only exit is the policy proxy. The OpenShell version is pinned exactly and the sandbox image by digest. The sandbox policy is defined on the host, outside anything the agent can write. Weak spots: Landlock runs in 'best_effort' mode so it can silently not apply on unsupported kernels, process limits are applied best-effort, and memory/CPU caps are optional.

- **S L3:** Hardened container: non-root user, read-only system paths, Landlock plus seccomp/no-new-privs via the pinned OpenShell runtime, network namespace with proxy-only egress. — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:62](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L62); [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:28](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L28); [Dockerfile:1614](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/Dockerfile#L1614); [nemoclaw-blueprint/blueprint.yaml:6-7](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/blueprint.yaml#L6-L7) (verified)
  - *To reach the next level:* Not kernel-separated (no microVM/gVisor).
- **C L3:** Every agent process runs inside the sandbox; there is no host execution path for model-driven commands. — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:62](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L62); [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:59](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L59) (verified)
  - *To reach the next level:* Landlock is best_effort, so one layer can fail open rather than failing closed.
- **D L3:** Sandboxing is always on and the policy lives on the host; the agent cannot request unsandboxed execution. — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:59](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L59); [nemoclaw-blueprint/blueprint.yaml:6-7](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/blueprint.yaml#L6-L7) (verified)
  - *To reach the next level:* Landlock degrades silently instead of requiring an explicit operator choice.
- **B L2:** No secrets in the sandbox and allowlisted egress, but the default allowlist includes uninspected npm pass-through and POST /** hosts, and memory/CPU caps are not set by default. — [nemoclaw-blueprint/policies/presets/npm.yaml:16-19](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/presets/npm.yaml#L16-L19); [scripts/lib/sandbox-rlimits.sh:8-9](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/scripts/lib/sandbox-rlimits.sh#L8-L9); [scripts/generate-openclaw-config.mts:1112](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/scripts/generate-openclaw-config.mts#L1112) (verified)
  - *To reach the next level:* Default egress includes broad channels and memory/CPU limits are optional.
- **Cap:** none

### C5 Untrusted input blast radius — 0.50 (high)

NemoClaw does not try to detect prompt injection; it relies on structural limits. Because egress is deny-by-default and unlisted hosts need operator approval, injected instructions cannot reach arbitrary servers, and the sandbox holds no raw credentials. But the default configuration still lets a hijacked agent read web content (web fetch is enabled) and send data out unattended through allowed endpoints such as POST to clawhub.ai or openclaw.ai, or the uninspected npm registry tunnel. Irreversible external actions are limited because messaging, email and GitHub integrations are off by default.

- **S L2:** Egress to new destinations needs operator approval, but egress to baseline/tier-allowed hosts is unattended regardless of what the agent has read. — [scripts/generate-openclaw-config.mts:1098](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/scripts/generate-openclaw-config.mts#L1098); [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:121-132](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L121-L132) (verified)
  - *To reach the next level:* No taint tracking: reading untrusted content does not disable or gate the remaining egress tools.
- **C L2:** The egress limit applies to every source equally since it is enforced at the network layer. — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:91](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L91) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished at all by NemoClaw; limits are not provenance-aware.
- **D L3:** The network policy is on by default and lives outside the sandbox, so content the agent reads cannot change it. — [nemoclaw-blueprint/blueprint.yaml:6-7](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/blueprint.yaml#L6-L7); [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:5-14](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L5-L14) (verified)
  - *To reach the next level:* Operator widening via policy commands is not warned in all paths.
- **B L1:** A hijacked agent can exfiltrate sandbox data unattended through allowed POST endpoints or the npm tunnel, though it cannot take irreversible external actions by default. — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:121-132](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L121-L132); [nemoclaw-blueprint/policies/presets/npm.yaml:16-19](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/presets/npm.yaml#L16-L19); [scripts/generate-openclaw-config.mts:1098](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/scripts/generate-openclaw-config.mts#L1098) (verified)
  - *To reach the next level:* Sessions that read untrusted content still have an unattended outbound channel.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.10 (high)

Everything that shapes the agent's future behaviour lives in a directory the agent can write: its config file, hooks, memory, cron jobs, extensions and skills under /sandbox/.openclaw. NemoClaw's own docs state it keeps no hash, seal or last-good copy of that config. So a single successful injection can persist itself as a hook, skill, cron job or config change that fires in later sessions. The saving grace is scope: the state belongs to one sandbox, and network policy and credentials are held outside it, so poisoned config cannot widen egress or steal keys.

- **S L0:** The agent can rewrite its own config, hooks, memory, skills and cron jobs with no validation or gate. — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:45](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L45); [Dockerfile:2166-2185](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/Dockerfile#L2166-L2185) (verified)
  - *To reach the next level:* Writes to memory and security-relevant config are not gated or validated.
- **C L0:** No memory or config path inside the sandbox is controlled by NemoClaw. — [Dockerfile:2166-2185](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/Dockerfile#L2166-L2185) (verified)
  - *To reach the next level:* No store is controlled.
- **D L1:** State is per-sandbox and file-permission isolated (0600 config, sandbox user), not shared across tenants; capped at one level above S because no write control exists. — [Dockerfile:2259](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/Dockerfile#L2259); [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:45](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L45) (verified)
  - *To reach the next level:* No write control on memory or config, and nothing stops the agent changing its own isolation-relevant config.
- **B L1:** Poisoned hooks, skills or cron jobs persist across the user's sessions and can trigger tool use. — [Dockerfile:2166-2185](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/Dockerfile#L2166-L2185) (verified)
  - *To reach the next level:* Persistent state is not reviewed or rollbackable short of recreating the sandbox.
- **Cap:** none

### C7 Third-party extensions — 0.17 (high)

What NemoClaw ships is well pinned: the sandbox image is pinned by digest, the OpenShell version exactly, and the OpenClaw runtime through a lockfile with integrity hashes. But the default Balanced tier opens npm, PyPI and Homebrew, and the baseline opens ClawHub, so the agent can install and run arbitrary packages, skills or plugins at runtime, chosen by the model, with no consent step that NemoClaw provides. Those extensions run as the same sandbox user with the agent's full state, though without raw provider credentials.

- **S L0:** Model-chosen package and skill installs from npm, PyPI, Homebrew and ClawHub are reachable by default with no pinning or verification. — [nemoclaw-blueprint/policies/presets/npm.yaml:16-19](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/presets/npm.yaml#L16-L19); [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:121-132](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L121-L132); [agents/openclaw/manifest.yaml:52](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/agents/openclaw/manifest.yaml#L52) (verified)
  - *To reach the next level:* Runtime-installed extensions are not pinned or integrity-checked.
- **C L1:** Only the NemoClaw-shipped image and runtime are pinned; runtime installs, skills and MCP bundles are not. — [nemoclaw-blueprint/blueprint.yaml:35](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/blueprint.yaml#L35); [agents/openclaw/openclaw-runtime/package-lock.json:21](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/agents/openclaw/openclaw-runtime/package-lock.json#L21) (verified)
  - *To reach the next level:* Most extension types are not verified.
- **D L1:** Nothing third-party is enabled at start beyond pinned plugins, but registries are reachable so the agent can add extensions without a consent prompt from NemoClaw. — [nemoclaw-blueprint/policies/tiers.yaml:23-31](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/tiers.yaml#L23-L31); [scripts/generate-openclaw-config.mts:863](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/scripts/generate-openclaw-config.mts#L863) (verified)
  - *To reach the next level:* Adding an extension does not show the exact package and require human consent.
- **B L1:** An installed extension runs as the sandbox user with access to all agent state, but the sandbox holds no raw provider keys and egress stays policy-limited. — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:62](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L62); [scripts/generate-openclaw-config.mts:1112](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/scripts/generate-openclaw-config.mts#L1112) (verified)
  - *To reach the next level:* Extensions are not sandboxed per extension with scoped credentials.
- **Cap:** C7-RCELOAD — In the default Balanced tier the agent can npm/pip/brew install and run arbitrary remote packages at the model's choice with no consent step.

### C8 Secrets & sensitive-data protection — 0.70 (high)

This is NemoClaw's strongest area. Provider and search keys are stored in the OpenShell gateway, never written into the sandbox; the agent config holds placeholder references that the egress proxy replaces only on the matching allowed endpoint, so secrets never reach the model or the agent's processes. The host CLI strips its environment before spawning children and redacts known secret patterns in logs and audit entries, and there is no crash or usage telemetry. Gaps: keys are long-lived, and the OpenClaw gateway token is readable by anything in the sandbox.

- **S L3:** Secrets live in the gateway store, never enter the sandbox or model context (opaque placeholders), and host logs are redacted. — [scripts/generate-openclaw-config.mts:1112](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/scripts/generate-openclaw-config.mts#L1112); [src/lib/security/redact.ts:190](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/src/lib/security/redact.ts#L190); [src/lib/adapters/openshell/provider-adapter-cli.ts:535](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/src/lib/adapters/openshell/provider-adapter-cli.ts#L535) (verified)
  - *To reach the next level:* Keys are not short-lived and there is no output scanning of agent responses.
- **C L3:** Model-bound context, sandbox processes, host subprocess env, logs and audit entries are covered. — [src/lib/subprocess-env.ts:97-112](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/src/lib/subprocess-env.ts#L97-L112); [src/lib/security/redact.ts:190](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/src/lib/security/redact.ts#L190); [src/lib/state/audit/operational.ts:37](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/src/lib/state/audit/operational.ts#L37) (verified)
  - *To reach the next level:* The in-sandbox OpenClaw gateway token is exposed to every sandbox process via a shell env file.
- **D L3:** No telemetry SDK ships; credential placeholdering and redaction are always on. — searched `rg -n -i -w "sentry|posthog|telemetry" --glob '!*.test.ts'` in `src/lib` → 8 hits (No crash-reporting/telemetry SDK; hits are a local OTLP test fixture, a README note on opt-in local OTLP, config verification, and GPU 'memory telemetry' messages.); [scripts/generate-openclaw-config.mts:1112](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/scripts/generate-openclaw-config.mts#L1112) (verified)
  - *To reach the next level:* Stored session transcripts are not encrypted or minimised by default.
- **B L2:** A host-side leak would expose long-lived provider API keys; the sandbox cannot leak them. — [nemoclaw-blueprint/blueprint.yaml:57](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/blueprint.yaml#L57) (verified)
  - *To reach the next level:* Keys are not short-lived or per-task.
- **Cap:** none

### C9 Audit & traceability — 0.40 (medium)

The detailed record of what the agent did is OpenClaw's own session transcript, stored inside the sandbox in a directory the agent can write and delete; NemoClaw only provides a command to export it. NemoClaw keeps a separate append-only host log, but it records only three operator actions (inference changes, config changes and token rotation), not agent tool calls. Network allow/deny decisions are visible in OpenShell's TUI and logs, which live outside the sandbox but belong to OpenShell, not this repository.

- **S L2:** Structured per-session JSONL transcripts (from OpenClaw) are exported by NemoClaw; NemoClaw's own audit log is structured but covers operator actions only. — [src/lib/actions/sandbox/sessions/export.ts:15-17](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/src/lib/actions/sandbox/sessions/export.ts#L15-L17); [src/lib/state/audit/operational.ts:23-28](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/src/lib/state/audit/operational.ts#L23-L28) (inferred)
  - *To reach the next level:* No actor attribution or tamper-evident storage for agent actions.
- **C L2:** Agent tool calls are covered by the OpenClaw transcript; NemoClaw adds operator config changes. — [src/lib/actions/sandbox/sessions/export.ts:15-17](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/src/lib/actions/sandbox/sessions/export.ts#L15-L17); [src/lib/state/audit/operational.ts:23-28](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/src/lib/state/audit/operational.ts#L23-L28) (inferred)
  - *To reach the next level:* Approvals/denials and extension activity are not recorded in a NemoClaw-owned record.
- **D L1:** The agent-action record lives inside the sandbox directory the agent can modify. — [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:45](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L45); [src/lib/actions/sandbox/sessions/export.ts:15-17](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/src/lib/actions/sandbox/sessions/export.ts#L15-L17) (verified)
  - *To reach the next level:* Records are not written by a component the model cannot control.
- **B L1:** Logging is best-effort and actions do not depend on records being written. — [src/lib/state/audit/operational.ts:19](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/src/lib/state/audit/operational.ts#L19) (verified)
  - *To reach the next level:* Records are not durable per action outside the agent's reach.
- **Cap:** none

### C10 Limits & kill switch — 0.33 (high)

NemoClaw sets a per-run agent timeout (600 seconds by default) and best-effort limits of 512 processes and 65,536 open files, and stopping or destroying the sandbox ends every process in it. There is no step cap or token/cost budget, and the timeout lives in the agent-writable config file, so the agent could raise it. OpenClaw cron jobs and heartbeats can keep the always-on agent working without a human, and inference spend is bounded only by the provider account.

- **S L1:** A wall-clock timeout per agent run plus process-count limits; no step or token/cost cap. — [scripts/generate-openclaw-config.mts:802](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/scripts/generate-openclaw-config.mts#L802); [scripts/generate-openclaw-config.mts:1015](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/scripts/generate-openclaw-config.mts#L1015); searched `rg -n "exec-approvals|askFallback|tools\.exec|maxTurns|maxIterations|maxSteps"` in `scripts/generate-openclaw-config.mts` → 0 hits (NemoClaw emits no OpenClaw exec-approval policy, step cap, or iteration cap into the generated agent config.) (verified)
  - *To reach the next level:* No iteration cap or token/cost budget enforced in code.
- **C L2:** The timeout applies to agent runs and the process limit to all sandbox processes. — [scripts/lib/sandbox-rlimits.sh:8-9](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/scripts/lib/sandbox-rlimits.sh#L8-L9); [Dockerfile:2166-2185](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/Dockerfile#L2166-L2185) (verified)
  - *To reach the next level:* Scheduled cron/heartbeat work and sub-agents are not counted against a shared budget.
- **D L1:** Defaults are sensible but stored in openclaw.json, which the agent can rewrite. — [scripts/generate-openclaw-config.mts:1015](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/scripts/generate-openclaw-config.mts#L1015); [nemoclaw-blueprint/policies/openclaw-sandbox.yaml:45](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/nemoclaw-blueprint/policies/openclaw-sandbox.yaml#L45) (verified)
  - *To reach the next level:* The model can raise its own timeout by editing its config.
- **B L1:** Destroying the sandbox stops everything, but until then an always-on agent can loop and spend without a ceiling. — [Dockerfile:2166-2185](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/Dockerfile#L2166-L2185); [scripts/generate-openclaw-config.mts:802](https://github.com/NVIDIA/NemoClaw/blob/e68bcacd808a916b6b7e07334d1256ce51f9c6ef/scripts/generate-openclaw-config.mts#L802) (verified)
  - *To reach the next level:* No spend ceiling or tight per-run cost limit.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Web fetch enabled by default and package/skill registries reachable (scripts/generate-openclaw-config.mts:1098, nemoclaw-blueprint/policies/tiers.yaml:23-31) · [B] sensitive data/systems: User conversations, workspace files and agent memory in /sandbox/.openclaw (nemoclaw-blueprint/policies/openclaw-sandbox.yaml:45) · [C] state change / egress: Shell and file writes in the sandbox plus unattended POST /** to clawhub.ai/openclaw.ai and L4 npm tunnel (nemoclaw-blueprint/policies/openclaw-sandbox.yaml:121-132, presets/npm.yaml:16-19) · Same default session? Yes

## Highest-impact improvements
1. Narrow the baseline clawhub.ai/openclaw.ai POST /** rules and default npm L4 pass-through so the default tier has no unattended exfiltration channel. — C5 B L1→L2, +0.050 before caps (Playbook 1)
2. Gate runtime package/skill installs behind operator consent (or default to the Restricted tier) and pin what is installed. — C7 S L0→L2, +0.150 before caps (Playbook 3)
3. Make security-relevant OpenClaw config (hooks, plugins, timeouts) root-owned or hash-verified at startup, leaving only memory/workspace writable. — C6 S L0→L2, +0.150 before caps (Playbook 2)
4. Ship an OpenClaw exec-approval policy in the generated config so shell commands and writes need per-call approval by default. — C2 C L1→L2, +0.075 before caps (Playbook 5)
5. Stream agent tool-call records to a host-side log outside the sandbox. — C9 D L1→L3, +0.100 before caps (Playbook 1 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Enforcement mechanisms (egress proxy, Landlock, seccomp, network namespace, operator approval TUI) are implemented in NVIDIA OpenShell, not this repository; they were credited based on the policy and pinned version NemoClaw ships and its own documentation, not by reading OpenShell source.
- Application-layer controls (tool approval, prompt-injection handling, session transcripts) belong to the upstream OpenClaw package, which was not reviewed; NemoClaw's own docs state it adds no protection there.
- Only the default OpenClaw agent was scored; Hermes and LangChain Deep Agents Code paths (which have their own approval/auto-approval controls) were not.
- The repository ships AGENTS.md/CLAUDE.md and agent skills addressed to coding agents; they are contributor workflow instructions, and no text attempting to steer safety reviewers was found.
