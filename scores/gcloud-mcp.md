# Defense-in-Depth Score: gcloud MCP

**Repo:** https://github.com/googleapis/gcloud-mcp · **Commit:** `238728f237ecb5f483199906b564a1767a0713f9` · **Reviewed:** 2026-10-03
**What it is:** Google Cloud MCP servers wrapping gcloud CLI, observability, storage
**Category:** Infrastructure & Ops
**Scored configuration:** The gcloud MCP server (packages/gcloud-mcp) launched over stdio as in the README (`npx -y @google-cloud/gcloud-mcp`, no --config), using the active gcloud account; sibling storage, observability and backupdr servers reviewed only for differences.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents no · external communication yes

## Score: 1.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | C1-SELFESC | **0.00** | Medium |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | Medium |
| C3 | Tool & action scoping | L1 | L2 | L1 | L0 | 0.28 | G2 | **0.25** | Medium |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | Medium |
| C6 | Memory, context & configuration integrity | L1 | L0 | L2 | L1 | 0.23 | — | **0.23** | Low |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | Low |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | Medium |
| C9 | Audit & traceability | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | Medium |
| C10 | Limits & kill switch | L1 | L0 | L0 | L0 | 0.07 | — | **0.07** | High |


As shipped, this server lets the model run almost any gcloud command with the user's full Google Cloud credentials, with no read-only mode, no confirmation, no risk annotations and no timeouts. The only built-in restriction is a short denylist of interactive commands (SSH, serial console), and it is not a strict boundary. The dominant risk is a prompt-injected model deleting resources, changing IAM, or exfiltrating data and tokens unattended; run it only under a narrowly scoped, impersonated service account. The sibling storage server in this repo is safer by default (no overwrite or delete tools unless enabled).

## Critical gaps
- The model can change IAM policy for the account it runs as, or mint service-account keys, because no IAM or auth command group is restricted by default (C1-SELFESC). (ASI03, T3; C1) — [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43); [packages/gcloud-mcp/src/gcloud_executor.ts:91-92](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/gcloud_executor.ts#L91-L92)
- Every command runs with the user's full ambient gcloud authority across all reachable projects (C1 blast radius L0). (ASI03; C1) — [packages/gcloud-mcp/src/gcloud_executor.ts:91-92](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/gcloud_executor.ts#L91-L92); [README.md:200-202](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/README.md#L200-L202)
- Model-chosen gcloud commands run unsandboxed as the user, with the user's credentials, and can deploy code that executes remotely (C4 blast radius L0). (ASI05, T11; C4) — [packages/gcloud-mcp/src/gcloud_executor.ts:91-92](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/gcloud_executor.ts#L91-L92); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:102](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L102)
- A hijacked session can leak data and credentials and take irreversible actions in GCP with no human step (C5-WORSTCASE). (ASI01, LLM01; C5) — [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:108-112](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L108-L112); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:102](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L102); [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43)
- The model can point gcloud's component manager at an arbitrary repository and install code that then runs as the user with their credentials (C7 blast radius L0; gcloud behaviour inferred). (ASI04, T17; C7) — [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43); [packages/gcloud-mcp/src/gcloud_executor.ts:91-92](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/gcloud_executor.ts#L91-L92)

## Criterion details

### C1 Identity & least privilege — 0.00 (medium)

The server runs every gcloud command with whatever account is active in the user's local gcloud installation, passing its full environment to the gcloud process, and nothing in the server narrows that authority. The model can also switch identity per call with global flags such as --impersonate-service-account or --account, and nothing stops it from changing IAM policy, including the policy that governs its own account. The README tells operators to impersonate a limited service account, but that is manual hardening outside the code.

- **S L0:** Every call uses the operator's ambient gcloud credentials with the full inherited environment; the server has no identity of its own and no per-request authorization check. — [packages/gcloud-mcp/src/gcloud_executor.ts:91-92](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/gcloud_executor.ts#L91-L92); [README.md:200-202](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/README.md#L200-L202); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:91](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L91) (verified)
  - *To reach the next level:* No deterministic authorization step maps calls to IAM permissions or narrows the credential; L1 needs at least a dedicated identity.
- **C L0:** The only check on the tool path is the command denylist, which is about interactive commands, not authorization; identity-switching global flags pass through unchecked. — [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:102](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L102) (verified)
  - *To reach the next level:* No authorization layer on any tool path.
- **D L0:** The default install acts with the full privileges of the active gcloud account; least privilege is left to the operator. — [README.md:200-202](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/README.md#L200-L202); [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43) (verified)
  - *To reach the next level:* Default is not read-only or role-scoped; least privilege requires manual hardening.
- **B L0:** A hijacked session holds the user's full GCP authority across every project the account can reach, including IAM writes and project deletion (gcloud command surface inferred from the absence of any restriction beyond the default denylist). — [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43); [packages/gcloud-mcp/src/gcloud_executor.ts:91-92](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/gcloud_executor.ts#L91-L92) (inferred)
  - *To reach the next level:* Nothing scopes the credential to one project, read-only operations, or a short lifetime.
- **Cap:** C1-SELFESC — The default denylist (index.ts:32-43, verified) does not block IAM commands, so the model can run `gcloud projects add-iam-policy-binding` or create service-account keys to widen its own authority (gcloud behaviour inferred).

### C2 Approval gates — 0.00 (medium)

The server exposes one tool, run_gcloud_command, that covers every read and every write in Google Cloud, with no risk annotations, no dry-run, and no read-only mode, so the host gets no signal to separate a harmless list from a project deletion. There is no server-side confirmation step. A wrongly approved or unapproved call can delete resources, change IAM, or deploy code, much of it irreversibly.

- **S L0:** A single generic tool mixes reads and writes and carries no readOnlyHint/destructiveHint annotations. — [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:36-41](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L36-L41); searched `rg -n 'readOnlyHint|destructiveHint|annotations'` in `packages` → 0 hits (No MCP tool annotations in any of the four servers.) (verified)
  - *To reach the next level:* Separate read and write tools with accurate annotations are required for L1-L2.
- **C L0:** The most powerful (and only) path, arbitrary gcloud commands, is exposed with nothing the host can key a gate on. — [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:36-41](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L36-L41); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:102](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L102) (verified)
  - *To reach the next level:* No risk signalling or server-side gate covers the tool.
- **D L0:** There is no approval or confirmation mechanism to turn on. — searched `rg -n -i 'confirm|approv|dry.?run|read.?only' --glob '!*.test.ts'` in `packages/gcloud-mcp/src` → 1 hits (Only hit is prompt text in the Gemini context file (init-gemini-cli.ts:46) asking the model to clarify ambiguity; not a control.) (verified)
  - *To reach the next level:* Approval support is absent; L1 needs a gate on by default.
- **B L0:** Irreversible actions (deleting Cloud SQL instances, buckets, VMs; IAM changes) are reachable with no undo provided by the server (gcloud command surface inferred). — [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:102](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L102) (inferred)
  - *To reach the next level:* No previews, dry-runs, or rollback for consequential operations.
- **Cap:** none

### C3 Tool & action scoping — 0.25 (medium)

The tool takes an argument array and runs gcloud without a shell, which rules out shell injection, and it parses each command with gcloud's own linter before checking it against a short default denylist of interactive commands (SSH, serial port, `interactive`, `meta`). Everything else, including IAM, secrets, deletes, deployments and local file copies, is allowed by default. Neither the denylist nor the operator allowlist is a strict boundary. An opt-in allowlist file is available but is off by default.

- **default configuration** (default; raw 0.28, cap G2 → 0.25) ← counted
  - **S L1:** A denylist of ten command groups, matched by prefix on the linter's parsed command path; arguments and global flags are not validated. — [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43); [packages/gcloud-mcp/src/denylist.ts:124-130](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/denylist.ts#L124-L130); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:102](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L102) (verified)
    - *To reach the next level:* Denylist only, and it is not a strict boundary; L2 needs typed validation of the arguments.
  - **C L2:** Every call of the single tool passes through the lint-and-ACL check before execution, and lint failure fails closed. — [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:80-83](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L80-L83); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:91](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L91) (verified)
    - *To reach the next level:* Coverage is capped one level above the weak denylist mechanism; flags such as --impersonate-service-account and --flags-file are never inspected.
  - **D L1:** All gcloud groups except the ten denied ones are enabled by default; operators can add deny entries via a config file. — [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43); [packages/gcloud-mcp/src/index.ts:110](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L110) (verified)
    - *To reach the next level:* Default set includes write, delete and IAM commands; L2 needs selectable tool groups and L3 a read-only default.
  - **B L0:** A misused tool reaches any gcloud command against any project the account can reach, plus local file reads and writes through `gcloud storage cp` (gcloud behaviour inferred). — [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43); [packages/gcloud-mcp/src/denylist.ts:102](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/denylist.ts#L102) (inferred)
    - *To reach the next level:* No project, resource, or quantity bound on what a call may touch.
- **opt-in allowlist via --config (allow: [...])** (alt; raw 0.42, cap G2 → 0.25)
  - **S L2:** An operator-written allowlist of command groups, matched on the parsed command path, denies anything not listed. — [packages/gcloud-mcp/src/index.ts:63-66](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L63-L66); [packages/gcloud-mcp/src/denylist.ts:103-110](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/denylist.ts#L103-L110) (verified)
    - *To reach the next level:* Enforcement is not a strict boundary, and identity-changing global flags are left unvalidated; L3 needs robust allowlist validation of all arguments.
  - **C L3:** The allowlist applies to every call of the only tool. — [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:91](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L91); [packages/gcloud-mcp/src/denylist.ts:49-54](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/denylist.ts#L49-L54) (verified)
    - *To reach the next level:* No central policy also covers flags and arguments; L4 needs every argument through one policy layer.
  - **D L0:** The allowlist is off unless the operator passes --config with an allow key. — [packages/gcloud-mcp/src/index.ts:63-66](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L63-L66); [packages/gcloud-mcp/src/denylist.ts:102](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/denylist.ts#L102) (verified)
    - *To reach the next level:* Not on by default.
  - **B L1:** Reach shrinks to the allowed groups, but any project and any impersonated identity remain reachable within them. — [packages/gcloud-mcp/src/index.ts:110](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L110) (inferred)
    - *To reach the next level:* No project or quantity bounds even with an allowlist.
- **Cap:** G2 — The command access-control check is not a strict boundary.

### C4 Code-execution isolation — 0.00 (medium)

Model-written arguments are run by the gcloud CLI as a child process of the server, as the same OS user, with the full environment and no sandbox. gcloud is a general tool: through it the model can copy local files anywhere, run scp, install components, and push code that runs remotely (VM startup scripts, Cloud Build, Cloud Run and Functions deploys) with the user's credentials. Nothing in the server isolates or limits any of this.

- **S L0:** Commands run as a same-user subprocess with no isolation primitive. — [packages/gcloud-mcp/src/gcloud_executor.ts:91-92](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/gcloud_executor.ts#L91-L92) (verified)
  - *To reach the next level:* No OS-level separation; L1 would need at least filtering of code-running subcommands.
- **C L0:** No execution path is sandboxed. — [packages/gcloud-mcp/src/gcloud_executor.ts:91-92](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/gcloud_executor.ts#L91-L92); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:102](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L102) (verified)
  - *To reach the next level:* No path goes through a sandbox.
- **D L0:** There is no sandbox to enable. — [packages/gcloud-mcp/src/gcloud_executor.ts:91-92](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/gcloud_executor.ts#L91-L92) (verified)
  - *To reach the next level:* Isolation absent by default.
- **B L0:** The gcloud process inherits the user's home directory, gcloud credential store, full environment and network, and can deploy code that runs remotely with the user's authority (gcloud behaviour inferred). — [packages/gcloud-mcp/src/gcloud_executor.ts:91-92](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/gcloud_executor.ts#L91-L92); [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43) (inferred)
  - *To reach the next level:* Credentials, home directory and network are all reachable; L1 needs at least credentials kept out of the execution context.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (medium)

Whatever gcloud prints, including log entries, resource metadata, labels and object contents that other people can write, goes back to the model as plain text with no marking of where it came from. The tool description itself contains directives to the model, such as preferring this tool over any other. There is no read-only or no-egress mode, so a hijacked model can read sensitive data, send it out (for example by copying it to an external bucket), and delete resources without any human step.

- **S L0:** Outputs are unlabelled plain text and the tool description contains directives to the model. — [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:48](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L48); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:108-112](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L108-L112) (verified)
  - *To reach the next level:* Plain outputs without directives (L1) and a structured split of content from metadata (L2) are missing.
- **C L0:** No untrusted source is distinguished; every gcloud output enters context the same way. — [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:108-112](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L108-L112) (verified)
  - *To reach the next level:* No source is treated differently.
- **D L0:** There is no provenance or mode control to enable. — searched `rg -n -i 'untrusted|provenance|structuredContent'` in `packages/gcloud-mcp/src` → 0 hits (verified)
  - *To reach the next level:* No control exists to be on by default.
- **B L0:** A hijacked model can both exfiltrate (e.g. `gcloud storage cp` to an attacker bucket, `auth print-access-token`) and take irreversible actions with no human involved (gcloud behaviour inferred). — [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:102](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L102) (inferred)
  - *To reach the next level:* Needs at least one of exfiltration or irreversible action to require approval.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.23 (low)

The server keeps no memory and reads no files from the workspace; its only configuration is an absolute-path file chosen by the operator. However, the model can persistently rewrite the user's own gcloud configuration through `gcloud config set`, for example redirecting API endpoints, switching the account, or setting service-account impersonation. Those changes survive the session and silently change every later gcloud call, by the agent and by the human. The only trace is the argument line in the server's stderr log.

- **S L1:** Writes to the persistent gcloud configuration are possible through `config set` and are logged (as tool input) but not validated (gcloud behaviour inferred; `config` is not denied). — [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:68](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L68) (inferred)
  - *To reach the next level:* Security-relevant gcloud properties are not protected from model writes; L2 needs config changes unable to alter security settings.
- **C L0:** No configuration path is controlled. — [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43) (inferred)
  - *To reach the next level:* No control on the gcloud config path.
- **D L2:** gcloud configuration is per OS user, enforced by the filesystem, but the model can switch or edit any named configuration. — [packages/gcloud-mcp/src/gcloud_executor.ts:91-92](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/gcloud_executor.ts#L91-L92) (inferred)
  - *To reach the next level:* The model can change the active configuration and its properties; L3 needs isolation the model cannot change.
- **B L1:** Poisoned configuration persists across the user's sessions and affects every later ungated gcloud call, e.g. an endpoint override would send bearer tokens elsewhere. — [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:102](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L102) (inferred)
  - *To reach the next level:* Changes are not reviewed, bounded to the session, or rolled back.
- **Cap:** none
- **Notes:** Operator config is loaded only from an absolute path given on the command line (index.ts:77-84), so workspace files cannot change the ACL.

### C7 Third-party extensions — 0.00 (low)

The server itself loads no plugins or remote code. But because it exposes the whole gcloud CLI and does not restrict the `components` group, the model can add a component repository at an arbitrary URL and install components from it, which gcloud then runs with the user's credentials. Nothing asks the user, pins versions, or confines what is installed. On package-manager installs of gcloud, the component manager may be disabled.

- **S L0:** Model-chosen installs via `gcloud components repositories add` / `components install` are reachable, with no verification by the server (gcloud behaviour inferred). — [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43); searched `rg -n 'components' --glob '!*.test.ts'` in `packages/gcloud-mcp/src` → 0 hits (Outside tests the server never mentions the components group, so it is neither denied nor gated.) (inferred)
  - *To reach the next level:* No pinning or integrity check of anything the model installs.
- **C L0:** No extension type is verified. — [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43) (inferred)
  - *To reach the next level:* Nothing verified.
- **D L0:** Installs happen at the model's request without a user consent step; gcloud prompts cannot reach a human because stdin is ignored, and --quiet skips them. — [packages/gcloud-mcp/src/gcloud_executor.ts:91-92](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/gcloud_executor.ts#L91-L92) (inferred)
  - *To reach the next level:* Needs at least an explicit consent before installation.
- **B L0:** An installed component runs as the same user with the full environment and gcloud credentials. — [packages/gcloud-mcp/src/gcloud_executor.ts:91-92](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/gcloud_executor.ts#L91-L92) (inferred)
  - *To reach the next level:* No separate process environment or credential scoping for installed code.
- **Cap:** none
- **Notes:** Surface exists only through gcloud's own component manager; components from Google's official repository are first-party, the risk is the model-added repository URL.

### C8 Secrets & sensitive-data protection — 0.05 (medium)

The server holds no keys of its own and leaves credentials in gcloud's credential store, which is a good start. But nothing keeps credentials or secret values out of the model's context: `gcloud auth print-access-token`, `secrets versions access` and similar commands return their output verbatim, and local credential files can be copied out with `gcloud storage cp`. There is no redaction anywhere; the server logs every command's arguments to stderr, but not outputs, and sends no telemetry.

- **S L0:** No masking or redaction exists on any path, and secret-returning gcloud commands are not restricted. — searched `rg -n -i 'redact|mask|secret|token' --glob '!*.test.ts'` in `packages/gcloud-mcp/src` → 0 hits; [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43) (verified)
  - *To reach the next level:* Needs at least masking on one path (L1).
- **C L0:** No path (logs, model-bound output, errors) is protected. — searched `rg -n -i 'redact|mask|secret|token' --glob '!*.test.ts'` in `packages/gcloud-mcp/src` → 0 hits; [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:108-112](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L108-L112) (verified)
  - *To reach the next level:* No path filtered.
- **D L1:** No telemetry; info-level stderr logging on by default includes full tool arguments, unredacted. — [packages/gcloud-mcp/src/utility/logger.ts:47-48](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/utility/logger.ts#L47-L48); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:68](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L68); [packages/gcloud-mcp/src/utility/logger.ts:118-123](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/utility/logger.ts#L118-L123) (verified)
  - *To reach the next level:* Redaction does not exist, so it cannot be on by default (L2 needs reasonable logging plus redaction).
- **B L0:** Long-lived, high-privilege credentials (gcloud refresh tokens, newly minted service-account keys) are reachable by the model through gcloud (gcloud behaviour inferred). — [packages/gcloud-mcp/src/gcloud_executor.ts:91-92](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/gcloud_executor.ts#L91-L92); [packages/gcloud-mcp/src/index.ts:32-43](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L32-L43) (inferred)
  - *To reach the next level:* Credentials reachable are not scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.38 (medium)

Each tool call writes a timestamped line to stderr with the full argument list before gcloud runs, and errors are logged. The record lacks the exit code or result, and denied commands are not logged at all. The logs go wherever the host stores the server's stderr, outside the server's control, and the agent's own gcloud access could in principle overwrite local files. There is no actor attribution beyond the tool name.

- **S L1:** Semi-structured stderr lines carry timestamp, tool name and arguments, but no result status. — [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:68](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L68); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:101](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L101); [packages/gcloud-mcp/src/utility/logger.ts:101](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/utility/logger.ts#L101) (verified)
  - *To reach the next level:* Result status is not recorded (L2 needs arguments, status and timestamps for every call).
- **C L2:** Every executed call is logged; denials return an error without any log line. — [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:101](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L101); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:92-98](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L92-L98) (verified)
  - *To reach the next level:* Denials are not recorded; L3 needs approvals/denials logged.
- **D L2:** On by default at info level and written to stderr for the host to store; the agent's gcloud access could still overwrite local log files. — [packages/gcloud-mcp/src/utility/logger.ts:47-48](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/utility/logger.ts#L47-L48); [packages/gcloud-mcp/src/utility/logger.ts:101](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/utility/logger.ts#L101) (inferred)
  - *To reach the next level:* Not written by a component the model cannot reach.
- **B L1:** Logging is best-effort console output; a lost log never blocks execution. — [packages/gcloud-mcp/src/utility/logger.ts:101](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/utility/logger.ts#L101); [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:101](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L101) (verified)
  - *To reach the next level:* No per-action durability guarantee or error surfacing on log failure.
- **Cap:** none

### C10 Limits & kill switch — 0.07 (high)

The server puts no bounds on its own work: gcloud processes run without a timeout or output-size cap, and calls are not rate- or concurrency-limited. The only limits are gcloud flags like --limit, which the model may choose to use. Streaming commands can run indefinitely, and on shutdown the server closes and exits without killing a running gcloud child.

- **S L1:** Only caller-chosen gcloud flags such as --limit bound output; the server enforces nothing. — [packages/gcloud-mcp/src/tools/run_gcloud_command.ts:59](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/tools/run_gcloud_command.ts#L59); searched `rg -n -i 'timeout|AbortController|maxBuffer' --glob '!*.test.ts'` in `packages/gcloud-mcp/src` → 2 hits (Both hits are setTimeout in utility/test_utils.ts (a test mock); no timeout or output cap on gcloud invocations.) (verified)
  - *To reach the next level:* No server-enforced timeout or output cap.
- **C L0:** No server-side limit applies to any call. — searched `rg -n -i 'timeout|AbortController|maxBuffer' --glob '!*.test.ts'` in `packages/gcloud-mcp/src` → 2 hits (Both hits are setTimeout in utility/test_utils.ts (a test mock); no timeout or output cap on gcloud invocations.); [packages/gcloud-mcp/src/gcloud_executor.ts:55-57](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/gcloud_executor.ts#L55-L57) (verified)
  - *To reach the next level:* Limits apply to nothing.
- **D L0:** Unlimited by default. — searched `rg -n -i 'timeout|AbortController|maxBuffer' --glob '!*.test.ts'` in `packages/gcloud-mcp/src` → 2 hits (Both hits are setTimeout in utility/test_utils.ts (a test mock); no timeout or output cap on gcloud invocations.) (verified)
  - *To reach the next level:* No default limits.
- **B L0:** A call waits for gcloud to close with no ceiling; shutdown exits without killing the child process. — [packages/gcloud-mcp/src/gcloud_executor.ts:55-57](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/gcloud_executor.ts#L55-L57); [packages/gcloud-mcp/src/index.ts:135-138](https://github.com/googleapis/gcloud-mcp/blob/238728f237ecb5f483199906b564a1767a0713f9/packages/gcloud-mcp/src/index.ts#L135-L138) (verified)
  - *To reach the next level:* No ceiling on runtime, output, or call rate.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: gcloud output (logs, metadata, object contents) returned verbatim (packages/gcloud-mcp/src/tools/run_gcloud_command.ts:108-112) · [B] sensitive data/systems: user's ambient gcloud credentials and all reachable GCP data (packages/gcloud-mcp/src/gcloud_executor.ts:91-92) · [C] state change / egress: any non-denied gcloud command, including deletes, IAM changes and storage cp (packages/gcloud-mcp/src/index.ts:32-43) · Same default session? Yes

## Highest-impact improvements
1. Harden enforcement of the command access check, and reject identity-changing global flags (--impersonate-service-account, --account, --configuration, --access-token-file, --flags-file) unless the operator allows them. — C3 S L1→L2, +0.075 before caps (Playbook 3, step 1)
2. Ship a read-only default (list/describe/get commands only) with writes enabled by an explicit operator flag, as the storage server already does. — C3 D L1→L3, +0.100 before caps (Playbook 3, step 1)
3. Split run_gcloud_command into read and write tools carrying accurate readOnlyHint/destructiveHint annotations so hosts can gate writes. — C2 S L0→L2, +0.150 before caps (Playbook 5, step 1)
4. Add a per-invocation timeout and output cap, and kill the gcloud child on timeout or shutdown. — C10 S L1→L2, +0.075 before caps (Playbook 3, step 3)
5. Log exit code and every denial as structured records alongside the arguments. — C9 S L1→L2, +0.075 before caps (Playbook 1, step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of commit 238728f only; no target code was installed, built or run.
- To confirm gcloud behaviour the review relied on, the locally installed Google Cloud SDK 530.0.0 (not from the target repo) was used for `--help` pages and to check the access-control logic; no GCP resources were contacted or changed.
- Scored the gcloud-mcp server that the README leads with. Sibling servers were reviewed only briefly and not scored: storage-mcp defaults to a safe mode without overwrite/delete tools (--enable-destructive-tools opt-in) but its upload tools read any local path; observability-mcp is list/get only; backupdr-mcp exposes create/delete tools. None of the four servers set MCP tool annotations.
- Many blast-radius ratings depend on gcloud's documented command surface (IAM, config set, components, storage cp); these are marked inferred.
- Windows executor path (windows_gcloud_utils.ts) reviewed only for environment handling.
- No reviewer-directed prompt injection found in the repo.
