# Defense-in-Depth Score: InkOS

**Repo:** https://github.com/narcooo/inkos · **Commit:** `8fc2ae57080b9821257dee3e37cc677e2b6f389a` (2.0.0) · **Reviewed:** 2026-10-04
**What it is:** Story-creation AI agent (Studio web app, CLI and TUI) for novels, short fiction, scripts, interactive fiction and translation.
**Category:** AI Assistants
**Scored configuration:** InkOS Studio on 127.0.0.1 with built-in Profiles, no env flags, provider keys saved through Studio.
**Agent surface (default):** code execution no · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions no · sub agents yes · external communication opt-in

## Score: 5.2 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L3 | L2 | L2 | 0.57 | — | **0.57** | High |
| C2 | Approval gates | L2 | L1 | L2 | L2 | 0.42 | — | **0.42** | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C6 | Memory, context & configuration integrity | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C9 | Audit & traceability | L2 | L3 | L2 | L4 | 0.68 | — | **0.68** | High |
| C10 | Limits & kill switch | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | High |

Controls where a risk surface exists: 3.17 / 8.0 (40%); 2 criteria scored SA (surface absent).

InkOS has no shell or code execution and routes every tool through one harness that logs each call before it runs and hides destructive deletion from the model. The main risk: a prompt-injected session can read the project's plaintext API-key file and send it to any URL through the material-ingestion tool, with no approval, because outbound fetches and all non-destructive writes run unattended. There are also no turn or spend limits.

## Critical gaps
- Unattended exfiltration chain: the read tool can open .inkos/secrets.json and ingest_material will GET any attacker URL without confirmation. (ASI01, LLM01, LLM02; C5) — [packages/core/src/harness/production-capabilities.ts:212](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/production-capabilities.ts#L212); [packages/core/src/llm/secrets.ts:17-22](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/llm/secrets.ts#L17-L22); [packages/core/src/harness/production-capabilities.ts:220](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/production-capabilities.ts#L220); [packages/core/src/materials/ingest.ts:124-134](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/materials/ingest.ts#L124-L134)

## Criterion details

### C1 Identity & least privilege — 0.57 (high)

InkOS runs as the local OS user but gives the model no shell and no generic file access: every tool goes through one harness runtime that checks the active Profile's capability list and each action's risk class before running it. File tools are confined to the project directory by a path-containment helper. The weak spot is that the project directory itself holds the Studio API keys in plaintext (.inkos/secrets.json), and the model's read tool is allowed to read any file under the project root, so the agent's own credentials are within its reach.

- **S L2:** Tools are domain-specific and confined to the project root; there is one static authority (OS user plus stored provider keys) for the whole run, and read and write share it. — [packages/core/src/harness/production-capabilities.ts:212](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/production-capabilities.ts#L212); [packages/core/src/agent/agent-tools.ts:3047-3052](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/agent-tools.ts#L3047-L3052); [packages/core/src/llm/secrets.ts:17-22](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/llm/secrets.ts#L17-L22) (verified)
  - *To reach the next level:* No per-tool credential scoping; the read tool can reach the project's own secrets file.
- **C L3:** Every model-visible tool is built by createCapabilityPiTools and executed through CreativeHarnessRuntime.executeAction, which enforces profile capability membership and the risk policy; worker sub-agents get no tools. — [packages/core/src/harness/runtime.ts:94-108](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/runtime.ts#L94-L108); [packages/core/src/harness/runtime.ts:221-234](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/runtime.ts#L221-L234); [packages/core/src/agent/worker-agent.ts:264](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/worker-agent.ts#L264) (verified)
  - *To reach the next level:* Single-user design: authorization is not evaluated against a requesting principal.
- **D L2:** Default read scope is the project; INKOS_AGENT_ALLOW_SYSTEM_READ=1 silently widens the read tool to the whole filesystem. — [packages/core/src/agent/agent-session.ts:842](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/agent-session.ts#L842); [packages/core/src/agent/agent-tools.ts:3047-3050](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/agent-tools.ts#L3047-L3050) (verified)
  - *To reach the next level:* Widening is a silent env var; default still includes write tools.
- **B L2:** If the profile gate fails, the agent can write anywhere in the project and read the stored LLM/image API keys; no other system credentials are held. — [packages/core/src/llm/secrets.ts:17-22](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/llm/secrets.ts#L17-L22); [packages/core/src/harness/production-capabilities.ts:211-215](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/production-capabilities.ts#L211-L215) (verified)
  - *To reach the next level:* Long-lived provider keys stored in the workspace are reachable.
- **Cap:** none

### C2 Approval gates — 0.42 (high)

InkOS has a real host-side confirmation path: creating new works goes through a propose_action card that the user must click in Studio (or a slash command), and the one destructive tool (delete latest chapter) is hidden from the model entirely. But the shipped Profiles set every recoverable write to execute without asking, and the URL-fetching ingest tool is classed as an ordinary recoverable write, so outbound requests and all edits to manuscripts and story state happen unattended. The approval card shows a model-written title and summary alongside the structured payload.

- **S L2:** propose_action returns a card with model-written title/summary/instruction plus a structured payload; execution only follows a host request with actionSource button/slash. — [packages/core/src/agent/agent-tools.ts:676-692](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/agent-tools.ts#L676-L692); [packages/core/src/agent/agent-session.ts:717-725](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/agent-session.ts#L717-L725) (verified)
  - *To reach the next level:* The approver sees model-written text, not a rendered diff of the exact executed call; no argument-level policy.
- **C L1:** Only actions with requiresConfirmation or destructive risk are gated; recoverable writes (chapter rewrites, truth-file writes, URL ingestion) run directly. — [packages/core/src/harness/runtime.ts:221-234](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/runtime.ts#L221-L234); [packages/core/src/harness/production-capabilities.ts:220](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/production-capabilities.ts#L220); [packages/core/src/agent/agent-session.ts:991-999](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/agent-session.ts#L991-L999) (verified)
  - *To reach the next level:* Network egress (ingest_material) and content-mutating writes bypass the gate.
- **D L2:** Built-in profiles set inferredMutation and explicitRecoverableMutation to execute; destructive confirm is a schema literal, and project .inkos/profiles files can only tighten it. — [packages/core/src/harness/builtin-profiles.ts:85-89](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/builtin-profiles.ts#L85-L89); [packages/core/src/harness/builtin-profiles.ts:94-104](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/builtin-profiles.ts#L94-L104); [packages/core/src/harness/contracts.ts:110-114](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/contracts.ts#L110-L114) (verified)
  - *To reach the next level:* Approval for recoverable writes is off by default.
- **B L2:** Artifact writes are revisioned and chapter deletes go to a trash folder, so most state changes are reversible; outbound HTTP requests are not. — [packages/core/src/harness/artifact-revisions.ts:38](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/artifact-revisions.ts#L38); [packages/core/src/state/chapter-delete.ts:73](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/state/chapter-delete.ts#L73) (verified)
  - *To reach the next level:* No preview or rate limit on egress; external requests are irreversible.
- **Cap:** none

### C3 Tool & action scoping — 0.45 (high)

The tool set is narrow by design: no shell, no generic HTTP client with methods, no SQL; tools are book-, chapter- and artifact-specific. File paths go through a containment check that is not a strict boundary. The URL ingestion tool accepts any http(s) URL with no host allowlist or block on localhost or cloud metadata addresses, which gives a hijacked agent an outbound GET channel and an SSRF primitive.

- **S L2:** Typed TypeBox schemas plus path containment (not a strict boundary), and a protocol-only check on URLs. — [packages/core/src/materials/ingest.ts:125-127](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/materials/ingest.ts#L125-L127); searched `rg -n -i '169\.254|isPrivateIp|isPrivateAddress|blockedHost|ssrf'` in `packages/core/src` → 2 hits (Both hits are in llm-endpoint-auth.ts (deciding whether an LLM endpoint needs an API key), not a fetch guard for ingest_material.) (verified)
  - *To reach the next level:* Path containment needs hardening and URLs are not checked against internal addresses or an allowlist.
- **C L2:** Most file tools use safeChildPath/safeBooksPath; ingest_material URL fetch and the model-supplied regex in grep are unvalidated. — [packages/core/src/materials/ingest.ts:124-134](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/materials/ingest.ts#L124-L134); [packages/core/src/agent/agent-tools.ts:3222](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/agent-tools.ts#L3222) (verified)
  - *To reach the next level:* URL and regex inputs are not validated.
- **D L2:** Tools are grouped by Profile capability lists, but every default profile includes write tools and the workspace capability always includes URL ingestion. — [packages/core/src/harness/production-capabilities.ts:208-222](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/production-capabilities.ts#L208-L222) (verified)
  - *To reach the next level:* No read-only default tool set.
- **B L1:** A misused ingest_material can GET any host, including localhost services, and write tools have full write within the project. — [packages/core/src/materials/ingest.ts:124-134](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/materials/ingest.ts#L124-L134) (verified)
  - *To reach the next level:* Egress is not scoped or bounded.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

No model-reachable path executes code or shell commands. The core agent package contains no process-spawning or eval calls outside tests, and the spawn calls in the Studio and CLI packages only launch the Studio server, open a browser, build the frontend, or run the user-invoked 'inkos update' command.

- **Structural absence:** searched `rg -n -S 'child_process|execSync|spawn\(|execFile|\beval\(|new Function\(|vm\.(run|Script)'` in `packages/core/src` → 2 hits (Both hits are in packages/core/src/__tests__/atomic-file-set.test.ts (test harness), not runtime code.); searched `rg -n -S 'child_process|execSync|spawn\(|execFile'` in `packages/studio/src packages/cli/src` → 10 hits (Hits are Studio launcher (spawn browser/server), auto vite build, and the explicit 'inkos update' command; none is reachable from a model tool call.)

### C5 Untrusted input blast radius — 0.05 (high)

InkOS reads plenty of content its user did not write: fetched web pages and PDFs, Tavily search results, uploaded files, imported chapters and canon. Nothing distinguishes that content from instructions once it is in context; tool descriptions merely say materials are 'reference only'. A hijacked session can, without any approval, read the project's stored API keys with the read tool and send them out in the query string of an ingest_material URL fetch. It cannot take destructive actions unattended, because deletion is hidden from the model and other writes are revisioned.

- **S L0:** No structural limit: the only defence is tool-description text telling the model materials are reference only. — searched `rg -n -i 'untrusted|taint|prompt.?injection'` in `packages/core/src` → 8 hits (All hits are 'uncertainties' in forecast code or a book-id test; no taint tracking or untrusted-content handling.); [packages/core/src/agent/agent-tools.ts:858-861](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/agent-tools.ts#L858-L861) (verified)
  - *To reach the next level:* No taint-aware gating of egress or writes after untrusted content is read.
- **C L0:** Fetched pages, uploads, imports and search results enter as ordinary tool results with no distinction. — [packages/core/src/materials/ingest.ts:143-147](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/materials/ingest.ts#L143-L147) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished.
- **D L0:** There is no control to configure. — searched `rg -n -i 'untrusted|taint|prompt.?injection'` in `packages/core/src` → 8 hits (All hits are 'uncertainties' in forecast code or a book-id test; no taint tracking or untrusted-content handling.) (verified)
  - *To reach the next level:* No untrusted-input control exists.
- **B L1:** Exfiltration is unattended (read .inkos/secrets.json, then ingest_material to an attacker URL), but irreversible destructive actions are not reachable without the user. — [packages/core/src/harness/production-capabilities.ts:211-215](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/production-capabilities.ts#L211-L215); [packages/core/src/llm/secrets.ts:17-22](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/llm/secrets.ts#L17-L22); [packages/core/src/harness/production-capabilities.ts:220](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/production-capabilities.ts#L220); [packages/core/src/harness/runtime.ts:221-234](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/runtime.ts#L221-L234) (verified)
  - *To reach the next level:* Egress after reading untrusted content does not require approval.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.38 (high)

InkOS is built around persistent story state: truth files, summaries, materials, a retrieval index and session transcripts are written by model-driven tools and re-injected into later sessions. Work artifacts are revisioned, so poisoned state can be inspected and rolled back, and project Profile files cannot loosen the destructive-action rule. However, SKILL.md instruction files in the project's skills/ and .agents/skills folders load silently and can replace built-in skills that Profiles activate automatically, and restored transcript system messages are re-inserted into the system prompt. In CLI mode, project-scoped configuration is additionally not integrity-protected; Studio, the scored mode, is not affected.

- **S L1:** Model writes to truth files are logged and revisioned but not validated for injected instructions; project-scope SKILL.md files load silently and can override built-in skill ids. — [packages/core/src/skills/external-loader.ts:102-108](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/skills/external-loader.ts#L102-L108); [packages/core/src/skills/builtin-loader.ts:32-34](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/skills/builtin-loader.ts#L32-L34); [packages/core/src/agent/agent-session.ts:1072-1074](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/agent-session.ts#L1072-L1074) (verified)
  - *To reach the next level:* No trust decision before project instruction files or override skills are loaded.
- **C L2:** Work artifacts (the main store) are revisioned through the harness; skills, transcripts and the materials index are not controlled. — [packages/core/src/harness/artifact-revisions.ts:38](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/artifact-revisions.ts#L38); [packages/core/src/agent/agent-session.ts:919-927](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/agent-session.ts#L919-L927) (verified)
  - *To reach the next level:* Auto-loaded skills and restored transcript context are not controlled.
- **D L2:** Storage is per project and per session id on a single-user local install. — [packages/core/src/interaction/session-transcript.ts:8-30](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/interaction/session-transcript.ts#L8-L30); [packages/core/src/agent/agent-session.ts:982](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/agent-session.ts#L982) (verified)
  - *To reach the next level:* No retention limits; isolation is directory-based only.
- **B L1:** Poisoned truth files or skills persist across the user's sessions and can steer tool use (including ingest_material egress). — [packages/core/src/skills/builtin-loader.ts:32-34](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/skills/builtin-loader.ts#L32-L34); [packages/core/src/harness/production-capabilities.ts:220](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/production-capabilities.ts#L220) (verified)
  - *To reach the next level:* Poisoned state can trigger ungated tool calls in later sessions.
- **Cap:** none
- **Notes:** CLI/TUI footnote: project-scoped configuration is not integrity-protected in CLI mode only; not applied as a cap because the scored Studio mode ignores env overlays.

### C7 Third-party extensions — 1.00 (high)

InkOS loads no third-party code at runtime: there is no MCP client, no plugin system that runs code, and no model-file loading. Skills are plain Markdown instructions (scored under C6). The only package install is the user-run 'inkos update' command for InkOS itself.

- **Structural absence:** searched `rg -n -i 'mcp|pickle|trust_remote_code|npx -y'` in `packages/core/src packages/studio/src/api` → 0 hits; searched `rg -n -S 'child_process|execSync|spawn\(|execFile|\beval\(|new Function\(|vm\.(run|Script)'` in `packages/core/src` → 2 hits (Both hits are in packages/core/src/__tests__/atomic-file-set.test.ts (test harness), not runtime code.)

### C8 Secrets & sensitive-data protection — 0.30 (high)

Provider API keys entered in Studio are saved in plaintext JSON inside the project at .inkos/secrets.json, written with default file permissions. The Studio API only returns whether a key exists, which is the single masking path. There is no redaction anywhere and the model's read tool can open the secrets file, so a key can end up in model context, transcripts and outbound requests. No telemetry SDK is present.

- **S L1:** Plaintext key file and env vars; the Studio API masks keys to a hasApiKey boolean. — [packages/core/src/llm/secrets.ts:17-22](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/llm/secrets.ts#L17-L22); [packages/studio/src/api/server.ts:1926](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/studio/src/api/server.ts#L1926); searched `rg -n -i 'redact|secretstr|maskSecret'` in `packages/core/src packages/studio/src/api` → 1 hits (Only hit is 'redacted_thinking' block-type handling; no secret redaction exists.) (verified)
  - *To reach the next level:* No redaction of logs, transcripts or model-bound content; no OS keychain.
- **C L1:** Only the Studio config API path avoids exposing keys. — [packages/studio/src/api/server.ts:1926](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/studio/src/api/server.ts#L1926); [packages/core/src/harness/production-capabilities.ts:211-215](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/production-capabilities.ts#L211-L215) (verified)
  - *To reach the next level:* Transcripts, model-bound tool results and logs are unprotected.
- **D L2:** No telemetry SDK ships; logging is not verbose by default, but nothing is redacted and the secrets file is written with default permissions. — searched `rg -n -i 'sentry|posthog|mixpanel|amplitude'` in `packages/core/src packages/studio/src packages/cli/src` → 0 hits; [packages/core/src/utils/atomic-file-set.ts:183](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/utils/atomic-file-set.ts#L183) (verified)
  - *To reach the next level:* Redaction does not exist to be always on.
- **B L1:** Long-lived provider API keys (LLM, image, search) are reachable by the model through the read tool. — [packages/core/src/llm/secrets.ts:17-22](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/llm/secrets.ts#L17-L22); [packages/core/src/harness/production-capabilities.ts:212](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/production-capabilities.ts#L212) (verified)
  - *To reach the next level:* Keys are long-lived and model-reachable.
- **Cap:** none

### C9 Audit & traceability — 0.68 (high)

Every tool call passes through the harness runtime, which writes an 'action-started' event with the parameters, risk class and request source to a SQLite episode ledger before the tool runs, then a completed, failed or cancelled event, plus an event whenever confirmation is required. A JSONL session transcript is kept as well. The ledger lives in the project's .inkos folder with no tamper protection, but the model has no tool that can write there.

- **S L2:** Structured events carry episode id, action id, parameters, source (agent vs explicit), risk and timestamps; trajectories tag main/subagent roles. — [packages/core/src/harness/runtime.ts:112-121](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/runtime.ts#L112-L121); [packages/core/src/llm/agent-trajectory.ts:3-11](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/llm/agent-trajectory.ts#L3-L11) (verified)
  - *To reach the next level:* No approver identity is recorded (only the request source), and there is no tamper-evident storage or standard export.
- **C L3:** All model tools go through executeAction, which logs starts, completions, failures and confirmation-required denials. — [packages/core/src/harness/runtime.ts:112-121](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/runtime.ts#L112-L121); [packages/core/src/harness/runtime.ts:98-107](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/runtime.ts#L98-L107) (verified)
  - *To reach the next level:* Config changes (Studio service settings) and credential use are not logged.
- **D L2:** On by default in .inkos/harness.sqlite and .inkos/sessions, outside the model's write tools, but writable by the agent process. — [packages/core/src/agent/agent-session.ts:982](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/agent-session.ts#L982); [packages/core/src/interaction/session-transcript.ts:8-30](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/interaction/session-transcript.ts#L8-L30) (verified)
  - *To reach the next level:* No component outside the agent process holds the record.
- **B L4:** The action-started append runs inside a transaction before capabilities.invoke; if it throws, the tool does not run. — [packages/core/src/harness/runtime.ts:112-121](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/runtime.ts#L112-L121); [packages/core/src/harness/episode-store.ts:58-99](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/harness/episode-store.ts#L58-L99) (verified)
- **Cap:** none

### C10 Limits & kill switch — 0.33 (high)

Each LLM request has a 5-minute deadline plus stream-idle deadlines, URL fetches time out after 15-20 seconds, and the user can abort a session, which propagates an abort signal into pipeline work. There is, however, no cap on agent turns, tool calls, tokens or cost per session, so a looping agent keeps running and spending until a human stops it.

- **S L1:** Per-request and per-fetch timeouts and a cooperative abort; no iteration, wall-clock or cost cap. — [packages/core/src/llm/provider.ts:73-81](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/llm/provider.ts#L73-L81); searched `rg -n -i 'maxTurns|maxSteps|maxIterations|max_turns|turnLimit|maxToolCalls'` in `packages/core/src` → 0 hits; [packages/core/src/agent/agent-session.ts:1466-1476](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/agent-session.ts#L1466-L1476) (verified)
  - *To reach the next level:* No step/turn cap or token/cost budget.
- **C L2:** The abort signal is passed to tools and pipeline sub-agent calls via runWithAgentContext; timeouts apply to LLM and fetch calls. — [packages/core/src/agent/agent-session.ts:1351](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/agent-session.ts#L1351); [packages/core/src/agent/agent-tools.ts:716-727](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/agent-tools.ts#L716-L727) (verified)
  - *To reach the next level:* Background confirmed tasks and sub-agent calls do not share a session budget.
- **D L1:** Only the 300s per-request timeout has a default (env-configurable); turns and spend are unlimited. — [packages/core/src/llm/provider.ts:73-81](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/llm/provider.ts#L73-L81) (verified)
  - *To reach the next level:* No sensible default ceiling on steps or spend.
- **B L1:** A runaway session has no ceiling on turns or spend; stop aborts the loop. — searched `rg -n -i 'maxTurns|maxSteps|maxIterations|max_turns|turnLimit|maxToolCalls'` in `packages/core/src` → 0 hits; [packages/core/src/agent/agent-session.ts:1466-1476](https://github.com/narcooo/inkos/blob/8fc2ae57080b9821257dee3e37cc677e2b6f389a/packages/core/src/agent/agent-session.ts#L1466-L1476) (verified)
  - *To reach the next level:* No per-run time or cost ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: ingest_material URL/PDF fetch, research_web results, uploads and imports (packages/core/src/materials/ingest.ts:128) · [B] sensitive data/systems: plaintext provider keys in .inkos/secrets.json readable by the read tool (packages/core/src/llm/secrets.ts:20) · [C] state change / egress: ungated arbitrary-URL GET and recoverable writes (packages/core/src/harness/production-capabilities.ts:220) · Same default session? Yes

## Highest-impact improvements
1. Deny .inkos/secrets.json, .env and other credential files in the read tool's path resolver. — C8 B L1→L2, +0.050 before caps (Playbook 4)
2. Block private, loopback and link-local addresses in ingest_material (re-checking redirects), and require confirmation for model-initiated URL fetches. — C3 S L2→L3, +0.075 before caps (Playbook 3)
3. Classify ingest_material as an egress action that needs confirmation once untrusted content has entered the session. — C5 S L0→L2, +0.150 before caps (Playbook 1)
4. Add a per-session cap on agent turns and tokens, enforced in the agent loop. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
5. Require an explicit trust step before project-scope skills/ or .agents/skills override built-in skills. — C6 S L1→L2, +0.075 before caps (Playbook 2)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Behaviour of the @mariozechner/pi-agent-core agent loop (e.g. any internal turn cap) and of the streamdown markdown renderer (remote image loading) was not examined because those dependencies are not vendored.
- CLI/TUI and daemon ('inkos up') modes were reviewed only where they differ materially; the scored mode is Studio.
- No text aimed at AI reviewers was found in the repository.
