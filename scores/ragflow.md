# Defense-in-Depth Score: RAGFlow

**Repo:** https://github.com/infiniflow/ragflow · **Commit:** `2400ca8eb51432b1d304266d0de9ca232d9b53fa` (nightly) · **Reviewed:** 2026-10-05
**What it is:** RAG engine with agent workflows, tools and code execution
**Category:** Agent Frameworks
**Scored configuration:** Self-hosted Docker Compose stack as shipped (docker/docker-compose.yml with docker/.env, Go backend via API_PROXY_SCHEME=go, default profiles without the optional sandbox profile): canvas agents with designer-selected tools, MCP servers, agentic RAG chat and shared agent links.
**Agent surface (default):** code execution opt-in · filesystem write no · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 2.9 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C2 | Approval gates | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C3 | Tool & action scoping | L2 | L2 | L3 | L1 | 0.50 | — | **0.50** | High |
| C4 | Code-execution isolation | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C7 | Third-party extensions | L1 | L1 | L2 | L2 | 0.35 | — | **0.35** | High |
| C8 | Secrets & sensitive-data protection | L0 | L1 | L1 | L1 | 0.17 | — | **0.17** | High |
| C9 | Audit & traceability | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L2 | 0.45 | — | **0.45** | High |


RAGFlow lets anyone who can build an agent give it web, SQL, email, HTTP and MCP tools that run on every model decision with no human approval, while the same sessions read uploaded documents, web pages and messages from visitors of shared agent links. The strongest controls are a pinned-IP SSRF guard on the URL tools, a tenant-scoped tool binding, and code execution that stays off until an operator deploys a sandbox; nothing limits what a prompt-injected agent does with the tools it was given. Credentials are stored in plaintext and agent tool calls leave no durable audit record. Treat every agent that combines document or web input with email, HTTP or MCP write tools as exposed to whoever can get text into its context.

## Critical gaps
- A hijacked agent can read private knowledge-base or database content and send it out or take irreversible sends with no human involved. (ASI01, LLM01, T6; C5) — [internal/agent/tool/email.go:97](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/email.go#L97); [internal/agent/component/agent.go:185-187](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/component/agent.go#L185-L187)

## Criterion details

### C1 Identity & least privilege — 0.40 (high)

Agents run with the authority of the tenant that built them: tool credentials (SMTP, SQL, API keys, MCP headers) are typed into the workflow by the designer and used for every caller, including anonymous visitors of a shared agent link. Code does scope data access: MCP servers are loaded only if they belong to the run's tenant, and agentic retrieval refuses dataset ids outside the conversation's bound scope. There is no per-requesting-user check on tools and no narrower credential for read versus write. The shipped deployment enables self-registration and bootstraps an administrator account at start-up.

- **S L2:** Tool authority is the tenant's static, designer-supplied credentials, but retrieval and MCP binding are scoped to the run's tenant and bound datasets in code. — [internal/agent/component/agent_mcp.go:47-50](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/component/agent_mcp.go#L47-L50); [internal/agentic_rag/helper.go:100-111](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agentic_rag/helper.go#L100-L111) (verified)
  - *To reach the next level:* No per-tool read-only credentials or per-request scoped tokens; one stored credential per tool serves every call.
- **C L2:** Built-in retrieval and MCP paths carry the tenant id, and sub-agents inherit the parent's tools; visitors of a shared agent act with the owner's tenant authority. — [internal/router/router.go:199-213](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/router/router.go#L199-L213) (verified)
  - *To reach the next level:* No authorization evaluated against the requesting principal on tool calls; shared-link visitors are not distinguished from the owner.
- **D L1:** The default install enables open self-registration and bootstraps an administrator account, so every registered user can build agents holding any credential they paste in. — [docker/.env:372](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/docker/.env#L372); [docker/docker-compose.yml:24-26](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/docker/docker-compose.yml#L24-L26) (verified)
  - *To reach the next level:* No read-only or minimal default role for agents; least privilege depends entirely on what each designer configures.
- **B L1:** A hijacked agent holds whatever the designer gave it: email sending, database read access, MCP write tools and the tenant's knowledge bases, across several systems. — [internal/agent/tool/email.go:97](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/email.go#L97); [internal/agent/tool/exesql.go:95](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/exesql.go#L95) (verified)
  - *To reach the next level:* Credentials are long-lived and span multiple systems; nothing confines them to one system or makes them short-lived.
- **Cap:** none

### C2 Approval gates — 0.05 (high)

There is no approval step anywhere on the agent tool path. The canvas agent hands its tool list straight to an automatic reason-act loop, so email sends, HTTP calls, SQL queries, code execution and MCP tools all run as soon as the model asks. A human-input node exists, but only as a workflow step the designer places, not as a gate on tool calls. An approval middleware exists in an unused internal package that no production code imports.

- **S L0:** The agent loop executes every tool call the model emits; no human or policy check sits between decision and execution. — [internal/agent/component/agent.go:185-187](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/component/agent.go#L185-L187); searched `rg -n -i 'approv'` in `internal/agent internal/agentic_rag internal/service internal/handler` → 3 hits (All three hits are prose (a regex comment and two prompt strings in a template); none is an approval check on tool calls.); searched `rg -n 'ragflow/internal/harness'` in `internal/agent internal/agentic_rag internal/service internal/handler internal/server cmd` → 0 hits (The internal/harness package, which contains an unused approval middleware, is not imported by any production path.) (verified)
  - *To reach the next level:* No per-call human approval showing the exact call.
- **C L0:** No tool, including email, MCP and code execution, crosses any gate. — [internal/agent/component/agent.go:697-706](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/component/agent.go#L697-L706) (verified)
  - *To reach the next level:* No gate on any tool path.
- **D L0:** There is no approval to turn on. — searched `rg -n -i 'approv'` in `internal/agent internal/agentic_rag internal/service internal/handler` → 3 hits (All three hits are prose (a regex comment and two prompt strings in a template); none is an approval check on tool calls.) (verified)
  - *To reach the next level:* Approval is not available as an option.
- **B L1:** A wrongly chosen action can send email to any address or call MCP and HTTP write endpoints irreversibly; code execution, when deployed, is container-scoped and SQL is filtered to reads. — [internal/agent/tool/email.go:97](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/email.go#L97); [internal/agent/tool/exesql.go:726](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/exesql.go#L726) (verified)
  - *To reach the next level:* No previews, dry-runs or rollback for external actions.
- **Cap:** none

### C3 Tool & action scoping — 0.50 (high)

URL-fetching tools resolve the host, reject private, loopback and metadata addresses, and pin the connection to the vetted IP, which closes the usual SSRF tricks; the HTTP component also refuses redirects. SQL is limited to reads by a keyword-level statement filter with a row cap, but email recipients are whatever the model chooses, code execution takes arbitrary scripts, and MCP arguments are passed through unchecked. Each agent only gets the tools its designer listed, unknown tool names are rejected, and the model cannot add tools.

- **S L2:** A strong resolved-IP SSRF guard covers URL tools, but SQL read-only enforcement is keyword based and email and code tools accept unrestricted recipients and scripts. — [internal/agent/tool/ssrf.go:74](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/ssrf.go#L74); [internal/agent/tool/ssrf.go:133](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/ssrf.go#L133); [internal/agent/tool/exesql.go:689](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/exesql.go#L689); [internal/agent/component/invoke.go:399](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/component/invoke.go#L399) (verified)
  - *To reach the next level:* No recipient allowlist for email and no parser-level SQL validation; general-purpose tools are not replaced by narrow ones.
- **C L2:** Most built-in network tools use the shared SSRF guard; MCP tool arguments are forwarded as raw JSON with no validation layer. — [internal/agent/tool/mcp.go:146](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/mcp.go#L146) (verified)
  - *To reach the next level:* No shared validation layer for extension (MCP) tools.
- **D L3:** A new agent has no tools; each agent node receives only the tools its designer lists, and unknown names fail the build. — [internal/agent/tool/registry.go:104-106](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/registry.go#L104-L106); [internal/agent/component/agent.go:697-698](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/component/agent.go#L697-L698) (verified)
  - *To reach the next level:* Model-side tool loading is impossible, but the per-node allowlist is the only scoping and C/D cannot exceed one level above S.
- **B L1:** A misused tool reaches any public host, any recipient, and any table readable by the configured database account. — [internal/agent/tool/email.go:97](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/email.go#L97) (verified)
  - *To reach the next level:* No quantity bounds on recipients or calls and no reversibility for sends.
- **Cap:** none

### C4 Code-execution isolation — 0.50 (medium)

Model-written Python or JavaScript runs only through a sandbox provider. The default provider points at a separate executor service that the shipped compose file does not start, so code execution fails until an operator deploys it, and there is no silent fallback to the host. When deployed, it runs each script in a stock Docker container with no network, 256 MB of memory and a 10-second timeout; seccomp is off by default and the manager itself runs privileged with the Docker socket. An administrator can instead pick a 'local' provider that runs code directly on the server with only a scrubbed environment. The agentic-RAG chat also has a JavaScript tool, but it runs in an embedded interpreter whose only capability is printing output.

- **S L2:** The default provider's isolation is a stock container with network disabled and seccomp off by default; hardening inside the externally built image could not be confirmed. — [docker/docker-compose-base.yml:199](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/docker/docker-compose-base.yml#L199); [docker/docker-compose-base.yml:207](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/docker/docker-compose-base.yml#L207); [internal/agentic_rag/tool_run_javascript.go:128-133](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agentic_rag/tool_run_javascript.go#L128-L133) (verified)
  - *To reach the next level:* No verified hardened profile (non-root, dropped capabilities, seccomp on, read-only root) or kernel-separated runtime in the default provider.
- **C L2:** CodeExec always goes through the provider manager and fails if no provider initializes; agentic-RAG JavaScript runs in a capability-free interpreter; the Browser component drives a local headless browser outside any sandbox. — [internal/agent/sandbox/manager_client.go:25-30](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/sandbox/manager_client.go#L25-L30); [internal/agent/component/browser.go:386](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/component/browser.go#L386) (verified)
  - *To reach the next level:* The browser path is not covered by the sandbox and the operator-selectable local provider has no warning.
- **D L2:** The sandbox service is an opt-in compose profile, but without it code does not run; an administrator can switch to host execution through the admin settings with no warning. — [internal/agent/sandbox/manager.go:281-285](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/sandbox/manager.go#L281-L285); [docker/.env:382](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/docker/.env#L382); [internal/agent/sandbox/local.go:18-22](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/sandbox/local.go#L18-L22) (verified)
  - *To reach the next level:* Switching to the unsandboxed local provider is a plain settings change rather than an explicit, loudly named operator flag.
- **B L2:** Runner containers get no network, a memory cap and a short timeout, but the manager that creates them is privileged with the host Docker socket, so a container escape is close to host root. — [docker/docker-compose-base.yml:183-190](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/docker/docker-compose-base.yml#L183-L190) (inferred)
  - *To reach the next level:* Runner pooling and image hardening are unverified, and the privileged manager raises the cost of an escape.
- **Cap:** none
- **Notes:** Remote sandbox providers (E2B, Aliyun, Tenki, UCloud) and an SSH provider are available through admin settings; they were not scored as alternatives because each is opt-in and capped at 0.50 under G1, which would not raise the criterion.

### C5 Untrusted input blast radius — 0.00 (high)

Nothing structural limits a hijacked agent. Retrieved document chunks, web results, crawler pages and MCP results all enter the model's context as ordinary tool output, with no taint tracking, no provenance-based gating and no approval before egress or sends. A single session can read untrusted documents, query private knowledge bases and databases, and send email or call HTTP and MCP endpoints. Shared agent links and optional chat-channel connectors let outside users talk to agents that hold the owner's tools, with no sender allowlist.

- **S L0:** No code limits what a hijacked agent can do; there is no injection handling at all. — searched `rg -n -i 'untrusted|prompt.injection'` in `internal/agent internal/agentic_rag` → 0 hits; [internal/agent/component/agent.go:185-187](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/component/agent.go#L185-L187) (verified)
  - *To reach the next level:* No approval or tool disabling once untrusted content has been read.
- **C L0:** Tool results, MCP results and tool descriptions enter context with the same standing as the user's instructions; MCP descriptions are passed to the model verbatim. — [internal/agent/tool/mcp.go:119-120](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/mcp.go#L119-L120); searched `rg -n -i 'allowed_users|allow_from|allowlist|whitelist'` in `internal/channels` → 0 hits (Chat-channel connectors have no sender allowlist.) (verified)
  - *To reach the next level:* No source is distinguished from principal input.
- **D L0:** There is no control to turn on. — searched `rg -n -i 'untrusted|prompt.injection'` in `internal/agent internal/agentic_rag` → 0 hits (verified)
  - *To reach the next level:* No control exists.
- **B L0:** A hijacked agent can read private knowledge-base and database content and send it out through email, HTTP or crawler query strings, and can take irreversible sends, all unattended; shared-link visitors make this reachable by outsiders. — [internal/agent/tool/email.go:97](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/email.go#L97); [internal/agent/tool/crawler.go:204](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/crawler.go#L204) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions both happen without a human.
- **Cap:** C5-WORSTCASE — Leak plus irreversible action is possible unattended in normal use.

### C6 Memory, context & configuration integrity — 0.20 (high)

Everything uploaded to a knowledge base persists and is retrieved into agent context for every user of that knowledge base, and agents can be configured to save each user input and answer into a memory store that later runs read back. Writes are not validated or reviewed. Isolation between tenants is enforced in queries, but within a tenant memory is shared across end users unless a user id happens to be supplied. Retrieved chunks carry document ids and can be inspected and deleted in the UI.

- **S L1:** Memory saves and document ingestion are automatic and unvalidated; retrieved content carries document or memory ids but is not treated differently from instructions. — [cmd/ragflow_server.go:1131](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/cmd/ragflow_server.go#L1131); [internal/agent/retrievalbridge/memory.go:62-64](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/retrievalbridge/memory.go#L62-L64) (verified)
  - *To reach the next level:* No gating, expiry or provenance-based handling of memory writes.
- **C L1:** Neither knowledge-base ingestion nor the memory store has write controls; only provenance ids are carried on retrieval. — [internal/agent/retrievalbridge/memory.go:62](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/retrievalbridge/memory.go#L62) (verified)
  - *To reach the next level:* No controlled store.
- **D L1:** Tenant id is required for memory retrieval, but the per-user filter is applied only when a user id is present. — [internal/agent/retrievalbridge/memory.go:62-64](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/retrievalbridge/memory.go#L62-L64) (verified)
  - *To reach the next level:* Per-user namespaces are not enforced by default.
- **B L0:** A poisoned document or memory entry persists across sessions and users and can steer tool calls in every later run that retrieves it. — [internal/agent/retrievalbridge/memory.go:62-64](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/retrievalbridge/memory.go#L62-L64) (verified)
  - *To reach the next level:* Poisoned content persists across users and drives tool use.
- **Cap:** none

### C7 Third-party extensions — 0.35 (high)

The only runtime extensions are MCP servers, which RAGFlow reaches over HTTP rather than launching locally, so no third-party code runs inside the RAGFlow process. A tenant user registers a server by URL, and an agent designer picks specific tools from it; the tool schemas captured at configuration time are what the agent uses later. Nothing pins or verifies the server's behaviour, and its tool descriptions go to the model verbatim. The server only receives its own configured headers plus whatever arguments the model sends.

- **S L1:** Servers are user-chosen URLs with no version pinning or integrity check; only the tool schema is snapshotted at configuration time. — [internal/agent/component/agent_mcp.go:68-73](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/component/agent_mcp.go#L68-L73) (verified)
  - *To reach the next level:* No pinning, integrity check or re-approval when a server's tools change.
- **C L1:** The schema snapshot applies to MCP, the only extension type. — [internal/agent/component/agent_mcp.go:50](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/component/agent_mcp.go#L50) (verified)
  - *To reach the next level:* No verification of any extension type.
- **D L2:** MCP servers are added explicitly by URL and tools selected per agent; nothing is enabled by default. — [internal/agent/component/agent_mcp.go:22](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/component/agent_mcp.go#L22) (verified)
  - *To reach the next level:* Adding a server does not present what it will do beyond a URL and tool list; C/D cannot exceed one level above S.
- **B L2:** A malicious server runs remotely and receives only its own headers and the model's arguments, but those arguments can carry any data in the agent's context. — [internal/agent/component/agent_mcp.go:68-70](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/component/agent_mcp.go#L68-L70) (verified)
  - *To reach the next level:* Arguments are not scoped, so other data in context can flow to the server.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.17 (high)

Provider API keys and tool credentials are stored as plaintext database columns and workflow parameters, with no masking type or general log filter. A few paths are careful: the HTTP access log drops query strings and tool errors scrub credential-named URL parameters. Sandbox runs get a scrubbed environment, and credentials are not placed in prompts. Verbose LLM request logging and per-tenant tracing exports are opt-in.

- **S L0:** Credentials are kept as plaintext columns and workflow parameters; scrubbing is limited to URL query parameters on two paths. — [internal/entity/tenant_llm.go:27](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/entity/tenant_llm.go#L27); searched `rg -n -i 'encrypt'` in `internal/entity internal/dao` → 1 hits (The single hit is a license blob column; no credential column is encrypted.) (verified)
  - *To reach the next level:* No encryption at rest or type-level masking of stored credentials.
- **C L1:** Only the access log and tool error URLs are scrubbed. — [internal/common/logger.go:464-474](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/common/logger.go#L464-L474); [internal/agent/tool/ssrf.go:253](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/ssrf.go#L253) (verified)
  - *To reach the next level:* No scrubbing of transcripts, error payloads or traces.
- **D L1:** Payload logging is opt-in, but nothing scrubs it when enabled and credentials are stored unprotected by default. — [docker/.env:358](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/docker/.env#L358) (verified)
  - *To reach the next level:* No always-on masking of logged payloads; C/D cannot exceed one level above S.
- **B L1:** A leak exposes long-lived provider, SMTP, database and MCP credentials, each scoped to one service. — [internal/agent/sandbox/local.go:471-473](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/sandbox/local.go#L471-L473) (verified)
  - *To reach the next level:* No short-lived or rotated credentials.
- **Cap:** none

### C9 Audit & traceability — 0.30 (high)

Each workflow node emits a structured finished event with its inputs, outputs (including the agent's tool calls) and timing, but these events are streamed to the client rather than stored. What persists is the conversation's user messages and answers in the database, plus run checkpoints that expire after 24 hours. Code-execution calls are logged only at debug level. There is no actor attribution beyond the session, no record of individual tool calls that survives the run, and nothing tamper-evident.

- **S L1:** Node events are structured but only streamed; durable records are conversation messages and ordinary server logs. — [internal/agent/canvas/scheduler.go:366](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/canvas/scheduler.go#L366); [internal/agent/tool/code_exec.go:166](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/code_exec.go#L166) (verified)
  - *To reach the next level:* No persisted structured record of every tool call with arguments and results.
- **C L1:** Answers are stored for all runs, but tool calls, MCP calls and sub-agent steps are not durably recorded. — [internal/agent/component/agent.go:1042](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/component/agent.go#L1042) (verified)
  - *To reach the next level:* Tool and extension calls are not covered.
- **D L2:** What is kept is written server-side where the agent's tools cannot reach it. — [cmd/ragflow_server.go:1520](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/cmd/ragflow_server.go#L1520) (verified)
  - *To reach the next level:* Records are not written by a component separated from the agent process.
- **B L1:** Run state expires after a day and missing events do not stop actions. — [cmd/ragflow_server.go:1534](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/cmd/ragflow_server.go#L1534) (verified)
  - *To reach the next level:* No durable per-action record.
- **Cap:** none

### C10 Limits & kill switch — 0.45 (high)

Canvas agents default to five reason-act rounds, sub-agents nest at most eight deep, workflow loops stop at 1,024 iterations, and code execution and MCP calls have timeouts. There is no token or spend cap and no run-level wall-clock limit, and each sub-agent gets a fresh round budget. The model may pass its own code-execution timeout, up to ten minutes. Cancelling a session sets a flag that is polled every half second and cancels the run's context, which interrupts the JavaScript interpreter and kills local code-execution process groups.

- **S L2:** Round caps and per-call timeouts are enforced in code; cancellation propagates through the run context. — [internal/agent/component/agent.go:178](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/component/agent.go#L178); [internal/agent/component/agent.go:862](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/component/agent.go#L862); [internal/agent/canvas/cancel.go:70-97](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/canvas/cancel.go#L70-L97); searched `rg -n -i 'cost|budget'` in `internal/agent/canvas` → 3 hits (Hits are test comments and a code comment about per-iteration lookup cost; no token or spend budget exists.) (verified)
  - *To reach the next level:* No token or cost cap.
- **C L2:** Limits apply to the top-level loop and tool calls; sub-agents get their own budget with only a depth cap. — [internal/agent/component/agent.go:41](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/component/agent.go#L41); [internal/agent/workflowx/loop.go:88](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/workflowx/loop.go#L88) (verified)
  - *To reach the next level:* Sub-agents do not count against the parent's budget.
- **D L1:** Defaults are moderate, but the model can raise the code-execution timeout per call up to 600 seconds. — [internal/agent/tool/code_exec.go:75](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/tool/code_exec.go#L75); [internal/agent/sandbox/provider.go:195](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/sandbox/provider.go#L195) (verified)
  - *To reach the next level:* The model can raise one of its own limits.
- **B L2:** Ceilings are moderate and stopping cancels in-flight work, but nested sub-agents and loops multiply model calls with no spend ceiling. — [internal/agent/sandbox/local.go:322-323](https://github.com/infiniflow/ragflow/blob/2400ca8eb51432b1d304266d0de9ca232d9b53fa/internal/agent/sandbox/local.go#L322-L323) (verified)
  - *To reach the next level:* No tight per-run cost or time ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Knowledge-base documents, web search and crawler results, MCP tool results, and visitors of shared agent links (internal/router/router.go:212-213) · [B] sensitive data/systems: Tenant knowledge bases, conversation memory, and designer-configured SQL databases (internal/agent/tool/exesql.go:95) · [C] state change / egress: Email tool with model-chosen recipients (internal/agent/tool/email.go:97), crawler and HTTP tools, and MCP tools · Same default session? Yes

## Highest-impact improvements
1. Add a per-call approval step (showing the exact recipient, URL, SQL or MCP arguments) for email, HTTP, MCP and code tools, on by default for shared agents. — C2 S L0→L3, +0.225 before caps (Playbook 5)
2. Once a session has retrieved documents, web pages or MCP output, force egress and send tools through approval. — C5 S L0→L3, +0.225 before caps (Playbook 1)
3. Persist every tool call (name, arguments, result status, session and requesting user) to a durable store instead of only streaming node events. — C9 S L1→L2, +0.075 before caps
4. Encrypt stored provider and tool credentials at rest and mask them in API responses and logs. — C8 S L0→L2, +0.150 before caps
5. Add a per-run token/cost budget shared with sub-agents and stop the model from raising the code-execution timeout. — C10 D L1→L2, +0.050 before caps

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The sandbox executor-manager and runner images (infiniflow/sandbox-*) are built outside this repository, so their gVisor and hardening claims could not be verified; ratings use only what the compose file and the Go client show.
- The legacy Python backend referenced in comments is not present at this commit; only the Go runtime, the web UI and the Docker deployment files were reviewed.
- internal/harness (an agent framework with approval middleware and shell tools) is not imported by any production path and was not scored.
- Ingestion parsers, data-source syncers and the admin UI were reviewed only where they touch agent tools, credentials or SSRF handling.
- AGENTS.md and other in-repo instructions were treated as data; no reviewer-directed content was found.
