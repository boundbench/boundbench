# Defense-in-Depth Score: Superagent SDK

**Repo:** https://github.com/superagent-ai/superagent · **Commit:** `c9e646271bf15cf7ea7b8e2de9de98008ea1099d` · **Reviewed:** 2026-10-04
**What it is:** Open-source AI safety SDK (TypeScript/Python), CLI and MCP server offering prompt-injection guard, PII redaction, and an LLM-agent repository scanner.
**Category:** Cybersecurity
**Scored configuration:** TypeScript SDK createClient() defaults (as used by the CLI and the stdio MCP server), with scan() at its default model and DAYTONA_API_KEY plus provider keys set.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents yes · external communication no

## Score: 2.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L0 | L1 | 0.12 | — | **0.12** | Medium |
| C2 | Approval gates | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C3 | Tool & action scoping | L2 | L1 | L1 | L2 | 0.38 | — | **0.38** | High |
| C4 | Code-execution isolation | L4 | L4 | L4 | L0 | 0.80 | — | **0.80** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | Medium |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L3 | 0.20 | C6-REPOCONFIG | **0.20** | Medium |
| C7 | Third-party extensions | L0 | L0 | L0 | L1 | 0.05 | C7-RCELOAD | **0.05** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C9 | Audit & traceability | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |


Guard and redact are single LLM calls with a well-hardened URL fetcher, but scan is an unattended agent: it installs opencode-ai@latest in a remote Daytona sandbox, points it at an untrusted repository, and gives it the operator's Anthropic and OpenAI API keys with unrestricted network. A poisoned repository can steal those keys, and nothing approves, logs, or time-limits the run. The sandbox keeps the operator's machine safe, but not their keys.

## Critical gaps
- Scan injects the operator's Anthropic/OpenAI API keys into the sandbox where an agent reads an untrusted repository with network access, so a poisoned repo can exfiltrate them. (ASI05, ASI03, LLM02; C4) — [sdk/typescript/src/client.ts:778-790](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L778-L790); [sdk/typescript/src/client.ts:809-811](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L809-L811)
- OpenCode runs with the scanned (untrusted) repository as its working directory, so repo-controlled agent config and instruction files can reconfigure the agent holding the operator's keys. (ASI06, ASI01; C6) — [sdk/typescript/src/client.ts:809-811](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L809-L811)
- Every scan installs and runs opencode-ai@latest unpinned, giving whatever is currently published the operator's LLM keys. (ASI04, LLM03; C7) — [sdk/typescript/src/client.ts:794](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L794)

## Criterion details

### C1 Identity & least privilege — 0.12 (medium)

The SDK reads the operator's own API keys from environment variables and, for repository scans, copies the operator's full Anthropic and OpenAI API keys into the remote sandbox where an autonomous agent reads the untrusted repository. Nothing narrows these keys to the task, issues short-lived credentials, or checks authority per request. Forwarding is limited to those two named variables rather than the whole environment, which is the only narrowing present. A hijacked scan agent holds billing-capable provider keys for two vendors.

- **S L0:** Scan forwards the operator's long-lived Anthropic and OpenAI keys unchanged into the sandbox environment. — [sdk/typescript/src/client.ts:778-790](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L778-L790); [sdk/typescript/src/providers/index.ts:111](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/providers/index.ts#L111) (verified)
  - *To reach the next level:* No scoped, per-task or short-lived credential; the agent uses the operator's full provider keys.
- **C L1:** Only two named keys are forwarded to the sandbox (not the whole environment), but no authorization layer checks any action. — [sdk/typescript/src/client.ts:778-790](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L778-L790) (verified)
  - *To reach the next level:* No authorization check on any tool path; the sandboxed agent's actions are unchecked.
- **D L0:** Forwarding happens unconditionally whenever the keys are set; there is no option to withhold them or supply a restricted key. — [sdk/typescript/src/client.ts:779-784](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L779-L784) (verified)
  - *To reach the next level:* No narrower default exists; least privilege would require the caller to unset their own keys.
- **B L1:** A stolen key grants full API use (and spend) on two separate LLM provider accounts. — [sdk/typescript/src/client.ts:778-790](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L778-L790) (inferred)
  - *To reach the next level:* Keys are not scoped to one system or to read-only use, and are not short-lived.
- **Cap:** none

### C2 Approval gates — 0.05 (high)

There is no human approval step anywhere. A scan call (from code, the CLI, or the MCP tool an AI host can invoke) provisions a paid remote sandbox, installs software, and runs an autonomous coding agent with shell and network access, with no confirmation. The only restraint is a prompt telling the scan agent to use read-only tools, which is not a control. The MCP server labels the scan tool read-only and idempotent, so hosts that auto-approve read-only tools will run it without asking.

- **S L0:** No approval mechanism exists; the scan agent is restrained only by prompt text, and the MCP scan tool is mislabelled readOnlyHint:true. — searched `rg -n -i 'approv|confirm'` in `sdk/typescript/src sdk/python/src cli/src mcp/src` → 0 hits (no approval or confirmation logic anywhere in SDK, CLI or MCP server); [sdk/typescript/src/prompts/scan.ts:26](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/prompts/scan.ts#L26); [mcp/src/index.ts:294-300](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/mcp/src/index.ts#L294-L300) (verified)
  - *To reach the next level:* No per-call approval or accurate risk signalling for the scan action.
- **C L0:** The most powerful path (scan: sandbox provisioning plus autonomous shell agent) runs with no gate. — [sdk/typescript/src/client.ts:809-811](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L809-L811); searched `rg -n -i 'approv|confirm'` in `sdk/typescript/src sdk/python/src cli/src mcp/src` → 0 hits (no approval or confirmation logic anywhere in SDK, CLI or MCP server) (verified)
  - *To reach the next level:* Scan and the shell commands run inside it bypass any gate because none exists.
- **D L0:** Nothing to enable; approval does not exist in any configuration. — searched `rg -n -i 'approv|confirm'` in `sdk/typescript/src sdk/python/src cli/src mcp/src` → 0 hits (no approval or confirmation logic anywhere in SDK, CLI or MCP server) (verified)
  - *To reach the next level:* Approval is not available even as an opt-in.
- **B L1:** Sandbox filesystem changes are discarded when the sandbox is deleted, but egress and spend on the operator's keys are irreversible. — [sdk/typescript/src/client.ts:822-829](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L822-L829); [sdk/typescript/src/client.ts:778-790](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L778-L790) (verified)
  - *To reach the next level:* No previews, dry-runs, or bounds on spend for external effects.
- **Cap:** none

### C3 Tool & action scoping — 0.38 (high)

The guard feature's URL fetcher is well built: it resolves hostnames, rejects private and internal addresses, pins the resolved address, re-validates every redirect, and caps size and time. The scan path is the opposite: the repository URL is only prefix-checked, and scan argument handling is not locked down. The scan agent itself gets OpenCode's general shell, edit and fetch tools. Everything is scoped to a throwaway remote sandbox.

- **S L2:** Strong SSRF allowlisting on URL fetch, but scan arguments get only prefix checks and their handling is not locked down. — [sdk/typescript/src/utils/safe-url-fetcher.ts:70-77](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/utils/safe-url-fetcher.ts#L70-L77); [sdk/typescript/src/utils/safe-url-fetcher.ts:204-207](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/utils/safe-url-fetcher.ts#L204-L207); [sdk/typescript/src/client.ts:748](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L748) (verified)
  - *To reach the next level:* Scan arguments (model, branch, repo) are not validated against allowlists; the agent gets general-purpose tools.
- **C L1:** Only the URL fetch path validates thoroughly; MCP zod schemas bound text length but the scan model/branch fields are free strings. — [mcp/src/index.ts:87-92](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/mcp/src/index.ts#L87-L92); [mcp/src/index.ts:320](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/mcp/src/index.ts#L320) (verified)
  - *To reach the next level:* Scan inputs and the sandboxed agent's tools are not validated.
- **D L1:** All three MCP tools, including the agent-running scan, are registered by default; scan only becomes usable when the operator sets DAYTONA_API_KEY. — [mcp/src/index.ts:302-314](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/mcp/src/index.ts#L302-L314) (verified)
  - *To reach the next level:* No read-only default set; scan cannot be disabled except by withholding the Daytona key.
- **B L2:** A misused scan agent has full shell and write inside one ephemeral remote sandbox. — [sdk/typescript/src/client.ts:787-790](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L787-L790); [sdk/typescript/src/client.ts:822-829](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L822-L829) (verified)
  - *To reach the next level:* No quantity bounds on what the sandboxed agent can do or fetch.
- **Cap:** none

### C4 Code-execution isolation — 0.80 (high)

All code execution happens in a remote, per-scan Daytona sandbox that is deleted afterwards; the SDK runs nothing on the operator's machine and has no fallback to local execution. That is a strong boundary that the model cannot switch off. What sits inside it is the problem: the operator's Anthropic and OpenAI API keys are injected into the sandbox environment and no network restriction is requested, so code from the scanned repository or a hijacked agent can read and send those keys out.

- **S L4:** Execution happens in a remote ephemeral sandbox service (Daytona), created per scan. — [sdk/typescript/src/client.ts:787-790](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L787-L790); [sdk/typescript/src/client.ts:809-811](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L809-L811) (verified)
- **C L4:** Every execution path (npm install, git clone, agent run) is a sandbox call; the host never spawns processes and there is no host fallback (missing Daytona key throws). — searched `rg -n -S 'child_process|execSync|spawn\(|subprocess|os\.system'` in `sdk/typescript/src sdk/python/src cli/src mcp/src` → 2 hits (both hits are prose inside the scan prompt (TS and Python); no host-side process execution exists); [sdk/typescript/src/client.ts:768-773](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L768-L773) (verified)
- **D L4:** Sandboxing is the only execution mode; the sandbox parameters are fixed in SDK code and nothing the model or scanned repo writes can change them. — [sdk/typescript/src/client.ts:787-790](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L787-L790) (verified)
- **B L0:** The operator's LLM API keys are placed in the sandbox environment and no egress restriction is configured. — [sdk/typescript/src/client.ts:778-790](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L778-L790) (verified)
  - *To reach the next level:* Secrets are in the sandbox environment and network egress is not restricted.
- **Cap:** none

### C5 Untrusted input blast radius — 0.05 (medium)

Scan exists to read untrusted repositories, and it hands that content to an autonomous agent with a shell, network access, and the operator's API keys, unattended. The only defence is a prompt telling the agent to stay read-only. Superagent's own guard classifier is not applied to the scanned content, and even as a product it is a detection layer, not a complete boundary. The agent's report is returned to the caller (or an MCP host model) as plain text with no marking that it was derived from attacker-controlled content.

- **S L0:** Nothing structurally limits a hijacked scan agent; the read-only rule is prompt text. — [sdk/typescript/src/prompts/scan.ts:26](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/prompts/scan.ts#L26); [sdk/typescript/src/client.ts:809-811](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L809-L811) (verified)
  - *To reach the next level:* No capability removal or approval once untrusted repo content has been read.
- **C L0:** The scanned repository and the agent's resulting output are not distinguished from trusted input; output goes back to the caller as plain text. — [sdk/typescript/src/client.ts:851-853](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L851-L853); [mcp/src/index.ts:323-330](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/mcp/src/index.ts#L323-L330) (verified)
  - *To reach the next level:* No untrusted source is handled; scan output carries no provenance.
- **D L0:** No mitigation exists to enable; guard is not wired into scan. — [sdk/typescript/src/client.ts:752-753](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L752-L753) (verified)
  - *To reach the next level:* No default-on limit for untrusted input.
- **B L1:** A poisoned repo can make the unattended agent exfiltrate the operator's LLM keys over unrestricted network (relies on OpenCode's non-interactive run allowing shell/fetch tools without prompts); no irreversible action on external systems is reachable because no other credentials are held. — [sdk/typescript/src/client.ts:778-790](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L778-L790); [sdk/typescript/src/client.ts:809-811](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L809-L811) (inferred)
  - *To reach the next level:* Exfiltration is possible without a human; no outbound channel is removed.
- **Cap:** none
- **Notes:** The guard is a detection layer, not a complete boundary.

### C6 Memory, context & configuration integrity — 0.20 (medium)

The project keeps no memory or persistent store of its own. However, the scan agent is started inside the freshly cloned, untrusted repository with no flags isolating it from project configuration, so files in that repository that OpenCode auto-loads (instruction files and project config that can add tools or MCP servers, change permissions, or redirect the model endpoint) can reconfigure the agent that holds the operator's keys. The damage is limited to one scan because each sandbox is deleted afterwards.

- **S L0:** OpenCode is run with the untrusted repo as its working directory and no isolation flags; OpenCode loads project AGENTS.md and opencode.json from the working directory (library behaviour, not verified in this repo). — [sdk/typescript/src/client.ts:809-811](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L809-L811); [sdk/typescript/src/client.ts:799](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L799) (inferred)
  - *To reach the next level:* Repo-controlled config is not blocked or gated by an explicit trust decision.
- **C L0:** No auto-loaded file path is controlled. — [sdk/typescript/src/client.ts:809-811](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L809-L811) (inferred)
  - *To reach the next level:* Neither instruction files nor project config are controlled.
- **D L1:** Each scan gets its own fresh sandbox, so nothing is shared across scans or users. — [sdk/typescript/src/client.ts:787-790](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L787-L790); [sdk/typescript/src/client.ts:822-829](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L822-L829); searched `rg -n -i 'memory|vector|sqlite|localStorage|AGENTS\.md'` in `sdk/typescript/src sdk/python/src cli/src mcp/src` → 0 hits (no memory store or instruction-file loader in the project's own code) (verified)
  - *To reach the next level:* Isolation is a side effect of sandbox lifetime, not an enforced namespace; capped by weak strength.
- **B L3:** Poisoned context lives only for one scan; the sandbox is deleted in a finally block. — [sdk/typescript/src/client.ts:822-829](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L822-L829) (verified)
  - *To reach the next level:* No human review step; within the scan it can still drive tool use.
- **Cap:** C6-REPOCONFIG — The scan agent runs inside the cloned untrusted repository, whose project files can configure tools, permissions and endpoints without any trust decision.

### C7 Third-party extensions — 0.05 (high)

Every scan runs `npm i -g opencode-ai@latest` inside the sandbox, so whatever version of that third-party agent is newest at that moment is installed and run with the operator's API keys, with no version pin, hash check, or notice to the user. A compromised release would get those keys on the next scan. The sandbox keeps it off the operator's machine.

- **S L0:** Third-party agent code is installed unpinned at @latest on every run. — [sdk/typescript/src/client.ts:794](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L794); [sdk/python/src/safety_agent/client.py:623](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/python/src/safety_agent/client.py#L623) (verified)
  - *To reach the next level:* No version pin or integrity check.
- **C L0:** The only runtime-loaded third-party component is unverified. — [sdk/typescript/src/client.ts:794](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L794) (verified)
  - *To reach the next level:* No extension type is verified.
- **D L0:** Installation happens automatically on every scan without showing what is installed. — [sdk/typescript/src/client.ts:794](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L794) (verified)
  - *To reach the next level:* No consent or disclosure of the installed package.
- **B L1:** The installed agent runs in the remote sandbox (not on the host) but holds the forwarded LLM keys. — [sdk/typescript/src/client.ts:778-790](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L778-L790); [sdk/typescript/src/client.ts:787-790](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L787-L790) (verified)
  - *To reach the next level:* Extension gets the operator's credentials; no scrubbed environment.
- **Cap:** C7-RCELOAD — Each scan downloads and executes the latest opencode-ai release without pinning or user consent.

### C8 Secrets & sensitive-data protection — 0.20 (high)

Keys come from environment variables and are not placed in prompts, but the scan path pushes the Anthropic and OpenAI keys into the sandbox where the model-driven agent can read them; the Python SDK's key handling is not locked down either. Usage telemetry is on by default and sends only token counts (with the Superagent key) to superagent.sh, with no opt-out. Error messages embed raw provider responses. There is no redaction of the SDK's own logs or errors.

- **S L1:** Secrets are read from env vars; forwarding to the sandbox is limited to two named keys, but the Python SDK's key handling is not locked down. — [sdk/typescript/src/client.ts:427](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L427) (verified)
  - *To reach the next level:* No masking, secret store, or redaction on any path.
- **C L1:** Only the subprocess-environment path is narrowed (named keys, not the full environment); errors include raw provider bodies. — [sdk/typescript/src/client.ts:778-790](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L778-L790); [sdk/typescript/src/providers/index.ts:172-175](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/providers/index.ts#L172-L175) (verified)
  - *To reach the next level:* Logs, error messages and agent output are not protected.
- **D L1:** Usage telemetry is on by default and content-free (token count only), with no opt-out. — [sdk/typescript/src/client.ts:444-453](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L444-L453) (verified)
  - *To reach the next level:* Telemetry is not opt-in.
- **B L0:** Long-lived provider keys are readable by the model-driven agent inside the sandbox. — [sdk/typescript/src/client.ts:778-790](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L778-L790) (verified)
  - *To reach the next level:* Keys are long-lived and reachable by the model's tools.
- **Cap:** none

### C9 Audit & traceability — 0.00 (high)

No record of what the scan agent did is kept. The SDK parses OpenCode's event stream and keeps only the final text and token counts, discarding the tool-call events, and the sandbox (with anything it logged) is deleted afterwards. There is no logging module, audit trail, or telemetry of actions.

- **S L0:** Tool-call events from the scan agent are dropped; only text and usage are kept. — [sdk/typescript/src/client.ts:851-866](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L851-L866); searched `rg -n -i 'audit|logger|opentelemetry'` in `sdk/typescript/src sdk/python/src cli/src mcp/src` → 5 hits (all 5 hits are prose in the scan prompt and an MCP tool description, not logging code) (verified)
  - *To reach the next level:* No structured record of tool calls.
- **C L0:** Nothing is recorded for any path. — searched `rg -n -i 'audit|logger|opentelemetry'` in `sdk/typescript/src sdk/python/src cli/src mcp/src` → 5 hits (all 5 hits are prose in the scan prompt and an MCP tool description, not logging code) (verified)
  - *To reach the next level:* No action recording at all.
- **D L0:** No logging exists to be enabled. — searched `rg -n -i 'audit|logger|opentelemetry'` in `sdk/typescript/src sdk/python/src cli/src mcp/src` → 5 hits (all 5 hits are prose in the scan prompt and an MCP tool description, not logging code) (verified)
  - *To reach the next level:* No default-on record.
- **B L0:** Sandbox deletion destroys any trace of the run. — [sdk/typescript/src/client.ts:822-829](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L822-L829) (verified)
  - *To reach the next level:* No durable record survives the run.
- **Cap:** none

### C10 Limits & kill switch — 0.20 (high)

The guard URL fetcher has firm size and 30-second limits, and MCP inputs are capped at 50,000 characters. The scan, the only agent loop, has no step, time, or cost limit: the sandbox command runs as long as the agent keeps going, spending on the operator's keys. Stopping cleanly deletes the sandbox, but if the process is killed the remote sandbox is left behind.

- **S L1:** Bounds exist only on URL fetches (30 s, 25 MB) and MCP input length; the scan agent has no step, time or cost cap. — [sdk/typescript/src/utils/safe-url-fetcher.ts:7-8](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/utils/safe-url-fetcher.ts#L7-L8); searched `rg -n -i 'timeout'` in `sdk/typescript/src/client.ts` → 1 hits (only hit is the guard cold-start fallbackTimeoutMs option; nothing bounds scan duration) (verified)
  - *To reach the next level:* No iteration, wall-clock or cost cap on the scan agent.
- **C L1:** Limits cover the fetch path only; the agent loop and sandbox commands are unbounded. — [sdk/typescript/src/client.ts:809-811](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L809-L811); searched `rg -n -i 'timeout'` in `sdk/typescript/src/client.ts` → 1 hits (only hit is the guard cold-start fallbackTimeoutMs option; nothing bounds scan duration) (verified)
  - *To reach the next level:* Limits do not apply to the scan agent or its commands.
- **D L1:** Fetch limits are on by default; nothing bounds scan by default. — [sdk/typescript/src/utils/safe-url-fetcher.ts:5-8](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/utils/safe-url-fetcher.ts#L5-L8) (verified)
  - *To reach the next level:* No sensible default ceiling for scans.
- **B L0:** A runaway scan can spend on the operator's keys with no ceiling; cleanup happens only if the SDK process survives to reach finally. — [sdk/typescript/src/client.ts:822-829](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L822-L829); [sdk/typescript/src/client.ts:809-811](https://github.com/superagent-ai/superagent/blob/c9e646271bf15cf7ea7b8e2de9de98008ea1099d/sdk/typescript/src/client.ts#L809-L811) (verified)
  - *To reach the next level:* No time or spend ceiling on a runaway scan.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: scanned repository cloned into the sandbox (sdk/typescript/src/client.ts:799) · [B] sensitive data/systems: operator ANTHROPIC/OPENAI API keys in sandbox env (sdk/typescript/src/client.ts:778-790) · [C] state change / egress: OpenCode agent with shell and unrestricted network in the sandbox (sdk/typescript/src/client.ts:809-811) · Same default session? Yes

## Highest-impact improvements
1. Stop forwarding provider keys into the sandbox; proxy model calls through a host-side or Superagent-side endpoint with a short-lived, spend-capped token. — C4 B L0→L3, +0.150 before caps (Playbook 4)
2. Pin opencode-ai to an exact version with an integrity hash (or bake it into a sandbox image). — C7 S L0→L3, +0.225 before caps (Playbook 3)
3. Add a wall-clock timeout and cost cap to the scan command and sandbox auto-delete on expiry. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
4. Validate the model/branch arguments against an allowlist. — C3 C L1→L2, +0.075 before caps
5. Persist the full OpenCode event stream (tool calls and arguments) in the scan result. — C9 S L0→L2, +0.150 before caps (Playbook 1 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- OpenCode's own behaviour (non-interactive tool permissions, auto-loading of project AGENTS.md/opencode.json) and Daytona's sandbox defaults (network, auto-stop) are third-party and were inferred, not verified; C5 B and C6 S/C are marked inferred.
- Scored the TypeScript SDK path; the Python SDK mirrors it. The remote superagent.sh services and guard models were not examined.
- No text aimed at AI reviewers was found in the repository.
