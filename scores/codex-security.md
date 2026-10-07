# Defense-in-Depth Score: Codex Security

**Repo:** https://github.com/openai/codex-security · **Commit:** `455544132aa06808b2a19cb48f01f6107d4c6b94` · **Reviewed:** 2026-10-03
**What it is:** OpenAI CLI and TypeScript SDK for finding, validating and fixing vulnerabilities
**Category:** Cybersecurity
**Scored configuration:** `codex-security scan <dir>` in standard mode with no flags or project config, on a Unix host using the shipped codex_security_scan permission profile.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents yes · external communication opt-in

## Score: 2.9 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L1 | L1 | 0.17 | — | **0.17** | High |
| C2 | Approval gates | L0 | L1 | L1 | L2 | 0.23 | C2-SELFAPPROVE | **0.23** | High |
| C3 | Tool & action scoping | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C4 | Code-execution isolation | L2 | L2 | L1 | L0 | 0.35 | G2 | **0.25** | Medium |
| C5 | Untrusted input blast radius | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C6 | Memory, context & configuration integrity | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | Medium |
| C7 | Third-party extensions | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | Low |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | Medium |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | Low |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |


Codex Security is a thin wrapper that drives Codex agents against code it is asked to review, under a filesystem profile that allows reading the whole machine and writing only to workspace roots. Execution approvals are decided by an automatic LLM reviewer rather than a person, and the agent's shell inherits the operator's environment, including unrelated cloud and API tokens. Its own tool-calling surface is Codex's, so most criteria depend on upstream behaviour; the wrapper adds careful path, executable and credential-home checks but no default cost or time ceiling in standard mode and, by repository policy, no secret redaction. Scan only repositories you trust, and start scans with only the credentials they need.

## Critical gaps
- Scan permission escalations are approved by a forced automatic LLM reviewer, not a human principal. (ASI09, ASI02; C2) — [sdk/typescript/src/api.ts:4594](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4594); [sdk/typescript/src/api.ts:4576](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4576)
- The model can obtain extra permissions from the automatic reviewer, so the sandbox can be widened at runtime without a human. (ASI05; C4) — [sdk/typescript/src/config.ts:56-57](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/config.ts#L56-L57); [sdk/typescript/README.md:2778-2781](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/README.md#L2778-L2781)
- The sandboxed scan shell inherits the operator's environment and can read the whole filesystem, so cloud and API tokens are reachable. (ASI05, ASI03; C4) — [sdk/typescript/src/api.ts:2712](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L2712); [sdk/typescript/src/api.ts:4601](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4601)

## Criterion details

### C1 Identity & least privilege — 0.17 (high)

The scan runs as the operator's own account. The Codex child process inherits the whole environment except OpenAI keys, and the scan profile lets the agent read the entire filesystem, so GitHub, AWS and Linear tokens and files such as SSH keys are within reach. Writes are limited to workspace roots, and the wrapper keeps its own model credentials in a private runtime home. Nothing narrows credentials per tool or per request.

- **S L0:** Authority is ambient: the child environment is the process environment minus OpenAI keys, and the profile grants read on the whole filesystem root. — [sdk/typescript/src/api.ts:2712](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L2712); [sdk/typescript/src/api.ts:4601](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4601) (verified)
  - *To reach the next level:* No dedicated, role-scoped identity; ambient cloud and token credentials are not withheld from the agent.
- **C L1:** Only the model-provider key is removed from subprocess environments; every other inherited variable reaches the shell, and external providers filter only competing provider keys. — [sdk/typescript/src/api.ts:4399-4413](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4399-L4413); [sdk/typescript/README.md:2786-2787](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/README.md#L2786-L2787) (verified)
  - *To reach the next level:* Subprocesses still receive the full ambient environment rather than a scoped identity.
- **D L1:** The default grants write only to workspace roots, but reads and the environment are broad; widening is limited to explicit operator flags such as the external sandbox for patching. — [sdk/typescript/src/api.ts:4602](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4602); [sdk/typescript/src/cli.ts:5288](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/cli.ts#L5288) (verified)
  - *To reach the next level:* Default is not near-minimal read-only identity; ambient read access and credentials are on by default.
- **B L1:** If authorization fails, the agent can read any local file and inherited token, while writes stay inside workspace roots; the draft PR and Linear paths are opt-in operator commands. — [sdk/typescript/src/api.ts:4601-4602](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4601-L4602); [sdk/typescript/src/cli.ts:6879](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/cli.ts#L6879) (verified)
  - *To reach the next level:* Credentials reachable are the operator's whole account across services and are long-lived.
- **Cap:** none

### C2 Approval gates — 0.23 (high)

Scan commands are approved by an automatic LLM reviewer: the wrapper forces the reviewer to auto_review and removes any override, so no person sees escalation requests in a normal scan. Per-call human approval does not exist in the default flow; the only operator control is setting approval_policy to never, which denies all requests. Consequential external actions (branch push, draft PR, Linear import) happen only when the operator passes explicit flags.

- **S L0:** The approver is an LLM reviewer forced on in code; no human approval step is offered for scan-time escalations. — [sdk/typescript/src/api.ts:4594](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4594); [sdk/typescript/src/api.ts:4576](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4576) (verified)
  - *To reach the next level:* No per-call human approval showing the exact command and arguments.
- **C L1:** Every Codex shell escalation goes through the same on-request policy and reviewer, but the reviewer is automatic and inherited MCP tools are governed upstream, not here. — [sdk/typescript/src/config.ts:173-181](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/config.ts#L173-L181); [sdk/typescript/src/api.ts:1910](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L1910) (verified)
  - *To reach the next level:* No verified human-gated or read-only-allowlisted path for all tools, including extensions.
- **D L1:** On by default as on-request, but the decision is the automatic reviewer; the operator can only tighten to never, and reviewer and sandbox overrides are stripped. — [sdk/typescript/src/config.ts:56-57](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/config.ts#L56-L57); [sdk/typescript/README.md:2778-2781](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/README.md#L2778-L2781) (verified)
  - *To reach the next level:* Approval by an authenticated human principal is not available as a default or an option.
- **B L2:** Default actions are reversible workspace writes; push and draft PR creation are explicit opt-in flags using a non-force push, and patching runs with approval never. — [sdk/typescript/src/cli.ts:6879](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/cli.ts#L6879); [sdk/typescript/src/cli.ts:6886](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/cli.ts#L6886) (verified)
  - *To reach the next level:* No rollback or checkpointing and no rate limits on approvals or consequential actions.
- **Cap:** C2-SELFAPPROVE — Approval of scan escalations is decided by a model reviewer (auto_review) forced on in code, not by a human principal.
- **Notes:** The cap trigger is verified in code; how the upstream reviewer decides is inferred from the README sentence on automatic review.

### C3 Tool & action scoping — 0.25 (high)

The wrapper does not define model-facing tools; the agent uses Codex's shell, so arguments are not validated per call. What the wrapper does constrain: the output directory must sit outside the repository, helper executables are resolved from PATH entries outside the target, git runs with fsmonitor disabled, and per-task profiles (policy generation, duplicate review, scan comparison) switch off MCP, plugins, web search and network. The default scan keeps the full tool set.

- **S L1:** Model shell commands pass through unvalidated, while the wrapper validates its own paths and executables with allowlist-style checks and filters config keys. — [sdk/typescript/src/runtime.ts:1519-1530](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/runtime.ts#L1519-L1530); [sdk/typescript/src/trusted-executable.ts:64](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/trusted-executable.ts#L64) (verified)
  - *To reach the next level:* No argument-level validation of model-chosen commands; general shell is not replaced by narrow tools.
- **C L1:** Validation applies to the wrapper's own helper processes, not to the tools the agent runs in the main scan. — [sdk/typescript/src/targets.ts:745](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/targets.ts#L745); [sdk/typescript/src/multiscan.ts:1069](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/multiscan.ts#L1069) (verified)
  - *To reach the next level:* Built-in agent tools and inherited extensions are not behind a shared validation layer.
- **D L1:** The default scan enables shell execution, workspace write and inherited extensions; several auxiliary tasks run with a reduced tool set but the main scan does not. — [sdk/typescript/src/api.ts:4674-4676](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4674-L4676); [sdk/typescript/src/config.ts:60-70](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/config.ts#L60-L70) (verified)
  - *To reach the next level:* Default scan tool set is not read-only and not per-task restricted.
- **B L1:** A misused shell reaches any readable file on the machine and can write anywhere under workspace roots. — [sdk/typescript/src/api.ts:4601-4602](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4601-L4602) (verified)
  - *To reach the next level:* Reach is not limited to a project and is not quantity-bounded.
- **Cap:** none

### C4 Code-execution isolation — 0.25 (medium)

Commands run inside Codex's native OS sandbox through a named profile that makes the filesystem read-only except workspace roots, and the wrapper refuses to start a scan on Unix if a sandboxed probe command fails. Overrides for the sandbox mode are stripped. However, escalations are approved by an automatic reviewer rather than a person, and the sandboxed shell carries the operator's full environment and can read the whole disk, so an escape or granted escalation reaches real credentials. Patching can opt out of the sandbox into an external one with a printed warning.

- **S L2:** The boundary is Codex's Seatbelt/Landlock/bubblewrap sandbox configured by a named profile; the implementation is upstream and was not reviewed, so strength is held at L2. — [sdk/typescript/src/api.ts:4596](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4596); [sdk/typescript/src/runtime.ts:2924](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/runtime.ts#L2924) (inferred)
  - *To reach the next level:* Sandbox primitive and network-denied default are inferred from upstream behaviour, not verified in this repository.
- **C L2:** Scan, validation, patch and verify turns all run under the sandbox and setup failure aborts on Unix; MCP servers inherited from user config and Windows are not covered by the probe. — [sdk/typescript/src/runtime.ts:2944](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/runtime.ts#L2944); [sdk/typescript/src/runtime.ts:2922](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/runtime.ts#L2922) (inferred)
  - *To reach the next level:* Inherited MCP processes and the Windows path have no verified sandbox or fail-closed probe.
- **D L1:** The sandbox is on by default and mode overrides are deleted, but on-request escalations are granted by an automatic reviewer without a human, so a model request can leave the boundary. — [sdk/typescript/src/api.ts:4575](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4575); [sdk/typescript/src/api.ts:4594](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4594) (verified)
  - *To reach the next level:* Escalation to unsandboxed or extra-permission execution is not a per-call human approval.
- **B L0:** The scan shell runs with the inherited environment (cloud and API tokens) and read access to the whole filesystem; only the policy task sets a core-only environment. — [sdk/typescript/src/api.ts:2712](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L2712); [sdk/typescript/src/api.ts:4665](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4665) (verified)
  - *To reach the next level:* Credentials are present in the sandbox environment and reads are not limited to the workspace.
- **Cap:** G2 — The model can request extra permissions that an automatic LLM reviewer grants for an operation, with no human, so the sandbox can be widened at runtime.
- **Notes:** Opt-in --external-sandbox for patching disables Codex enforcement and relies on a container; the Compose files drop all capabilities, set no-new-privileges, a seccomp profile and a non-root user, but that is a separate deployment mode and was not scored as the default.

### C5 Untrusted input blast radius — 0.25 (high)

A security scanner reads untrusted code by design. Defence rests on prompts that tell the agent to treat repository text as data and on delimiting untrusted JSON in auxiliary turns, plus the sandbox. Nothing tracks taint or forces human approval once untrusted content is read, and the reviewer that grants escalations is itself an LLM. Auxiliary tasks such as scan comparison and policy drafting are structurally limited, but the main scan is not.

- **S L1:** Delimiting plus instructions to ignore embedded directives; the no-tool comparison turn is structural but the main scan relies on prompts and the sandbox. — [sdk/typescript/src/scan-comparison.ts:963](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/scan-comparison.ts#L963); [AGENTS.md:6](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/AGENTS.md#L6) (verified)
  - *To reach the next level:* No code-enforced rule that disables egress and state changes or forces a human once untrusted content is read.
- **C L1:** Untrusted treatment is explicit for supplied context and comparison JSON, while repository files, imported Linear issues and GitHub alerts share the model context with the operator's instructions. — [sdk/typescript/src/linear.ts:35-39](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/linear.ts#L35-L39); [plugins/codex-security/skills/security-scan/SKILL.md:24](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/plugins/codex-security/skills/security-scan/SKILL.md#L24) (verified)
  - *To reach the next level:* Tool results, imported issues and repository SECURITY.md guidance are not separated from instructions in code.
- **D L1:** Instructions apply by default; no knob exists to disable them, but they are prompt text that content the agent reads can contradict. — [sdk/typescript/src/api.ts:4033](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4033) (verified)
  - *To reach the next level:* Nothing the agent reads is prevented from reconfiguring the behaviour, because it is not enforced in code.
- **B L1:** If a scan is hijacked, whole-disk reads and inherited tokens are reachable and permission escalations are granted by an LLM reviewer, so exfiltration is not human-gated; writes remain workspace-scoped and reversible. — [sdk/typescript/src/api.ts:4594](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4594); [sdk/typescript/src/api.ts:4601](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L4601) (verified)
  - *To reach the next level:* Exfiltration channels are not provably closed and escalation approval is not human.
- **Cap:** none
- **Notes:** The shipped GitHub Actions example limits runs to same-repository, non-draft, non-dependabot pull requests and warns against pull_request_target, so the public-trigger cap was not applied.

### C6 Memory, context & configuration integrity — 0.50 (medium)

Nothing from the target repository can add tools, hooks or settings: the project config file is loaded only when the operator names it, the agent works in an output directory required to sit outside the repository, and there is no dotenv loading. Repository SECURITY.md guidance and the operator's saved findings do enter later scans as context. Findings, dedupe groups and severity assessments persist locally and can feed the patch command, which runs in workspace-write.

- **S L2:** Repository config cannot change security settings and the working directory is outside the repository, but repository SECURITY.md guidance and saved results load silently as context. — [sdk/typescript/src/cli.ts:9518](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/cli.ts#L9518); [sdk/typescript/src/api.ts:1361](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L1361); searched `rg -n -i dotenv` in `sdk/typescript/src sdk/typescript/package.json` → 0 hits (No dotenv or .env loading in the SDK source.) (verified)
  - *To reach the next level:* No workspace-trust prompt or provenance tagging for repository policy files and persisted findings.
- **C L2:** Config and working directory are controlled; auto-loaded repository guidance and retrieval-style stores such as saved findings are not. — [sdk/typescript/src/api.ts:1908](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L1908); [sdk/typescript/src/security-policy.ts:800](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/security-policy.ts#L800) (verified)
  - *To reach the next level:* Instruction files and persisted stores are not all controlled or provenance-tagged.
- **D L2:** State is per scan directory and per local user by default; the findings service can opt into an all-repositories scope. — [sdk/typescript/src/finding-retrieval.ts:4-5](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/finding-retrieval.ts#L4-L5) (inferred)
  - *To reach the next level:* Namespace separation for the findings store is by query scope rather than storage, and retention limits are not on by default.
- **B L2:** Poisoned findings or guidance persist across the user's scans and can influence a later patch run, which works only in the workspace. — [sdk/typescript/src/cli.ts:7689-7690](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/cli.ts#L7689-L7690) (inferred)
  - *To reach the next level:* Persistence is not human-reviewed or rollbackable before it can influence actions.
- **Cap:** none

### C7 Third-party extensions — 0.30 (low)

The wrapper loads no third-party code on its own: its plugin ships inside the package and the Codex runtime is pinned exactly. Extensions come only from the operator's own Codex configuration, which the scan inherits, including MCP servers that deep-scan workers are documented not to disable. The wrapper does not pin, hash-check or re-approve those, and they run as the same user with the full environment.

- **S L1:** Inherited extensions are user-chosen and unpinned by the wrapper; only the bundled Codex runtime and plugin are pinned. — [sdk/typescript/package.json:77-78](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/package.json#L77-L78); [SECURITY.md:33](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/SECURITY.md#L33) (inferred)
  - *To reach the next level:* No version pinning, integrity check or re-approval for user-configured MCP servers or plugins.
- **C L1:** Only the bundled plugin is pinned; MCP servers and provider auth commands from config are not verified. — [sdk/typescript/src/config.ts:111-135](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/config.ts#L111-L135) (inferred)
  - *To reach the next level:* Other extension types have no verification.
- **D L2:** Nothing third-party is enabled by default and the repository cannot add extensions; adding one is an operator config change that shows no exact command at scan time. — [sdk/typescript/src/cli.ts:337](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/cli.ts#L337) (verified)
  - *To reach the next level:* Adding an extension does not display the exact package, command and permissions.
- **B L1:** Inherited MCP servers run as separate processes of the same user; the plugin's own MCP launch forwards a long allowlist of credential variables including AWS and provider keys. — [plugins/codex-security/.mcp.json:21](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/plugins/codex-security/.mcp.json#L21) (inferred)
  - *To reach the next level:* No scrubbed environment or per-extension sandbox.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.20 (medium)

The model-provider key is withheld from the agent's environment and the credential home and SQLite store are created private, with unusually careful ownership and ACL checks. Beyond that, other inherited secrets reach every subprocess and the model can read any file. There is no redaction of diagnostics or logs: contributor policy in AGENTS.md explicitly forbids adding it, and raw agent reasoning is shown by default.

- **S L1:** Secrets come from env vars or the Codex credential store with private 0700/0600 storage, but no masking or redaction exists anywhere in the SDK source. — [AGENTS.md:51-53](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/AGENTS.md#L51-L53); searched `rg -n -i redact` in `sdk/typescript/src` → 0 hits (Zero redaction code in SDK source.) (verified)
  - *To reach the next level:* No type-level masking, log filters or redaction before logs or model-bound messages.
- **C L1:** Only credential storage and the OpenAI key in the child environment are protected; logs, diagnostics, transcripts and model-bound messages are not. — [sdk/typescript/src/runtime.ts:222](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/runtime.ts#L222); [sdk/typescript/src/runtime.ts:1186](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/runtime.ts#L1186) (verified)
  - *To reach the next level:* Logs, telemetry and model-bound messages have no protection.
- **D L1:** Raw agent reasoning is displayed by default and analytics are an upstream setting the wrapper only passes through as an override; content scope of telemetry is not verified here. — [sdk/typescript/src/config.ts:62](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/config.ts#L62) (inferred)
  - *To reach the next level:* Content-free opt-in telemetry and always-on redaction are not shipped.
- **B L0:** Long-lived tokens in the operator's environment are reachable by the model and every subprocess because only OpenAI keys are filtered. — [sdk/typescript/src/api.ts:2712](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/api.ts#L2712); [sdk/typescript/README.md:2786-2787](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/README.md#L2786-L2787) (verified)
  - *To reach the next level:* Reachable keys are not scoped, short-lived or withheld.
- **Cap:** none

### C9 Audit & traceability — 0.50 (low)

Each scan thread, including sub-agent threads, is recorded by Codex as local session files that the wrapper reads for cost tracking and saved scan logs, and scan results are kept in a private SQLite workbench. The record sits outside the repository but is written by the same host process tree, with no tamper evidence or export. Logging is best effort by project policy.

- **S L2:** Structured per-thread session records exist in the runtime Codex home; their per-tool-call detail and approver attribution are inferred from upstream behaviour. — [sdk/typescript/src/cost.ts:203](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/cost.ts#L203) (inferred)
  - *To reach the next level:* No verified separation of agent, requesting principal and approver, and no tamper evidence or OpenTelemetry export.
- **C L2:** Scan, validation and sub-agent threads are located through saved thread ids; approvals and configuration changes are not separately recorded by the wrapper. — [sdk/typescript/src/scan-logs.ts:19](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/scan-logs.ts#L19) (inferred)
  - *To reach the next level:* Approvals, denials, config changes and credential use are not recorded by the wrapper.
- **D L2:** Records live under the runtime state directory outside the repository, but the same user process tree can alter them. — [sdk/typescript/src/runtime.ts:1186](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/runtime.ts#L1186) (inferred)
  - *To reach the next level:* Not written by a component the model cannot influence, and not protected from the operator-level user.
- **B L2:** Logging is explicitly best effort, so failures never stop work; session files are written per action by Codex. — [AGENTS.md:10](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/AGENTS.md#L10) (verified)
  - *To reach the next level:* High-risk actions do not wait on their audit record.
- **Cap:** none

### C10 Limits & kill switch — 0.25 (high)

Deep scans have shipped ceilings (40 discovery runs, 96 hours, 4 workers, stop after consecutive errors), while the default standard scan has no wrapper-enforced time or cost limit; a dollar cap exists but is optional. Interrupts forward SIGINT or SIGTERM to the child and force a kill after one second. Sub-agent concurrency is capped by a Codex setting, but limits are not enforced per tool call.

- **S L1:** Iteration and time limits exist only in deep mode and the cost cap is optional; halt kills the Codex child after a one-second grace. — [sdk/typescript/src/deep-scan-defaults.ts:7-8](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/deep-scan-defaults.ts#L7-L8); [sdk/typescript/src/scan-settings.ts:119](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/scan-settings.ts#L119) (verified)
  - *To reach the next level:* Standard mode lacks step, wall-clock and cost caps enforced in code by default.
- **C L1:** Limits cover the top-level deep loop; sub-agent threads are bounded by a concurrency setting but share no verified budget. — [sdk/typescript/src/config.ts:68](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/config.ts#L68) (verified)
  - *To reach the next level:* Sub-agents and spawned processes do not demonstrably count against a shared budget.
- **D L1:** Defaults exist but a 96-hour ceiling is very large, and the plugin's MCP tool timeout is about 97 hours. — [plugins/codex-security/.mcp.json:55](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/plugins/codex-security/.mcp.json#L55) (verified)
  - *To reach the next level:* Defaults are not tight and no hard ceiling prevents configuration from raising them.
- **B L1:** Ceilings are measured in days and standard mode has none, though stop forwards signals and kills the child. — [sdk/typescript/src/cli.ts:1679](https://github.com/openai/codex-security/blob/455544132aa06808b2a19cb48f01f6107d4c6b94/sdk/typescript/src/cli.ts#L1679) (verified)
  - *To reach the next level:* No tight per-run time and cost ceilings and no provider-side spend budget.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Target repository contents, imported issues and alerts enter the scan prompt (sdk/typescript/src/api.ts:1908) · [B] sensitive data/systems: Whole-filesystem read and inherited process environment (sdk/typescript/src/api.ts:4601, 2712) · [C] state change / egress: Workspace writes plus LLM-reviewed permission escalations (sdk/typescript/src/api.ts:4594, 4602) · Same default session? Yes

## Highest-impact improvements
1. Run the scan shell with a core-only environment (as policy generation already does) and withhold cloud and API tokens. — C4 B L0→L2, +0.100 before caps
2. Offer per-call human approval showing the exact command instead of forcing the LLM reviewer. — C2 S L0→L3, +0.225 before caps
3. Enforce default wall-clock and cost ceilings in standard mode as deep mode already does. — C10 S L1→L3, +0.150 before caps
4. Narrow the scan profile's root read access to the target, plugin and system paths instead of the whole filesystem. — C1 S L0→L2, +0.150 before caps
5. Add redaction of credential-shaped text in diagnostics and logs, reversing the current contributor policy. — C8 S L1→L3, +0.150 before caps

## Re-audit log
- No changes.

## Limitations
- Static source review only at the pinned commit; nothing was installed, built, or run.
- The agent loop, sandbox implementation, auto-review approver, session transcripts and analytics live in the upstream openai/codex dependency (pinned 0.162.0-alpha.7), which was not reviewed; ratings that depend on it are marked inferred.
- The bundled plugin skill prompts and the native Rust/MCP-app code under plugins/codex-security were sampled, not read exhaustively, and Windows sandbox paths were not examined.
- Scored the standard-mode CLI default; deep mode, patch, publish, bulk-scan containers and the findings service are footnoted but not separately scored.
- No attempt to steer reviewers was found; AGENTS.md addresses contributors (including a rule against adding secret redaction) and was treated as data.
