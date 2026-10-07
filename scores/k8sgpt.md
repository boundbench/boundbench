# Defense-in-Depth Score: K8sGPT

**Repo:** https://github.com/k8sgpt-ai/k8sgpt · **Commit:** `635d9b122318a41ada1bfcf9153801365fdcaf77` · **Reviewed:** 2026-10-03
**What it is:** CNCF Kubernetes diagnostics CLI/operator that scans clusters and explains issues with LLMs; ships MCP server
**Category:** Infrastructure & Ops
**Scored configuration:** The CLI as the README Quick Start leads: `k8sgpt auth add` (default OpenAI backend, key stored in the user config), then `k8sgpt analyze --explain` with no other flags: the operator's current kubeconfig context, all core analyzers, file cache on, --anonymize off. The serve, MCP and operator modes are footnoted, not scored.
**Agent surface (default):** code execution no · filesystem write no · network egress no · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents no · external communication no

## Score: 5.8 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L0 | L3 | 0.30 | — | **0.30** | High |
| C2 | Approval gates | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C3 | Tool & action scoping | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L4 | L3 | L4 | L3 | 0.88 | — | **0.88** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L2 | 0.35 | — | **0.35** | High |
| C7 | Third-party extensions | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |
| C8 | Secrets & sensitive-data protection | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C9 | Audit & traceability | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |

Controls where a risk surface exists: 2.77 / 7.0 (40%); 3 criteria scored SA (surface absent).

As a CLI, k8sgpt is a fixed pipeline, not a tool-using agent: hard-coded read-only analyzers collect cluster problems, the model explains each one in text, and nothing the model says is executed. A hijacked model can only mislead the operator. The main posture gaps are elsewhere: the CLI uses the operator's full kubeconfig with no narrowing, keeps no record of what it sent to the AI provider, and stores the provider key in plaintext. The score does not cover `k8sgpt serve --mcp`, which hands an MCP host tools that read any Secret and rewrite cache and analyzer endpoints. Assess that mode separately before using it.

## Critical gaps
- None: no criterion is capped and no blast radius is at the worst level.

## Criterion details

### C1 Identity & least privilege — 0.30 (high)

The CLI uses whatever Kubernetes credentials are in the operator's kubeconfig (in-cluster service account first, otherwise the current context), so it typically runs with cluster-admin-level authority and never narrows it. What limits the damage is the design: every Kubernetes request is a hard-coded list or get call in the analyzers, the model never sees or uses the credentials, and nothing on the analyze path writes to the cluster. A hijacked model therefore cannot use the identity at all, but k8sgpt itself still reads cluster-wide with the full credential, including Secret objects it fetches to check that they exist.

- **S L1:** Ambient kubeconfig credential with no narrowing, but the request set is fixed read-only code (no model-originated API calls). — [pkg/kubernetes/kubernetes.go:46-60](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/kubernetes/kubernetes.go#L46-L60); searched `rg -n '\.(Create|Update|Patch|Delete|DeleteCollection)\(' --glob '!*_test.go'` in `pkg/analyzer pkg/analysis pkg/ai pkg/kubernetes cmd/analyze` → 0 hits (No Kubernetes write call anywhere on the analyze path; all 22 matches without the glob are fake-client setup in _test.go files.) (verified)
  - *To reach the next level:* No dedicated or role-scoped identity: the operator's current context is used as-is, with no read-only impersonation or token downscoping.
- **C L1:** Every analyzer shares the same ambient client; there is no authorization check, only the fact that each call is a fixed get/list. — [pkg/analysis/analysis.go:97-100](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analysis/analysis.go#L97-L100) (verified)
  - *To reach the next level:* No authorization layer that every request passes through; coverage rests on code shape rather than a check.
- **D L0:** Default is the kubeconfig current context (or the pod's service account), whatever its privilege. — [cmd/root.go:86-87](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/cmd/root.go#L86-L87); [pkg/kubernetes/kubernetes.go:54-58](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/kubernetes/kubernetes.go#L54-L58) (verified)
  - *To reach the next level:* No least-privilege default; running with a read-only identity requires the operator to prepare one and pass --kubeconfig/--kubecontext.
- **B L3:** Only hard-coded read calls run against one cluster, the model cannot issue requests, and secret objects are fetched only to test existence; the reads are cluster-wide. — searched `rg -n '\.(Create|Update|Patch|Delete|DeleteCollection)\(' --glob '!*_test.go'` in `pkg/analyzer pkg/analysis pkg/ai pkg/kubernetes cmd/analyze` → 0 hits (No Kubernetes write call anywhere on the analyze path; all 22 matches without the glob are fake-client setup in _test.go files.); [pkg/analyzer/ingress.go:163](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analyzer/ingress.go#L163) (verified)
  - *To reach the next level:* Credentials are long-lived ambient kubeconfig, not minutes-lived and task-scoped; reads span all namespaces by default.
- **Cap:** none
- **Notes:** Serve/MCP mode (not scored) exposes get-resource and list-resources for Secrets to the MCP host; its credential handling is not locked down. The Helm chart's ClusterRole grants get/list/watch on all resources in all API groups (charts/k8sgpt/templates/role.yaml).

### C2 Approval gates — 1.00 (high)

In the scored CLI flow the model only returns text. It has no tools, and k8sgpt never acts on what the model says: the answer is stored in a local cache and printed. All Kubernetes calls on this path are read-only, so no consequential action exists that would need approval. The MCP server mode (not scored) is different: an MCP host can call a config tool that persists new custom-analyzer endpoints and remote cache buckets with no confirmation step.

- **Structural absence:** searched `rg -n -i 'tool_calls|toolcall|functioncall|tool_choice|tooluse|function_call'` in `pkg/ai pkg/analysis cmd/analyze` → 0 hits (No tool or function calling in any AI client or the analysis pipeline.); searched `rg -n '\.(Create|Update|Patch|Delete|DeleteCollection)\(' --glob '!*_test.go'` in `pkg/analyzer pkg/analysis pkg/ai pkg/kubernetes cmd/analyze` → 0 hits (No Kubernetes write call anywhere on the analyze path; all 22 matches without the glob are fake-client setup in _test.go files.); [pkg/ai/iai.go:71-72](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/ai/iai.go#L71-L72); [pkg/analysis/analysis.go:695-703](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analysis/analysis.go#L695-L703)

### C3 Tool & action scoping — 1.00 (high)

The model is given no tools, so it never chooses arguments, paths, URLs or queries. Which analyzers run, the namespace and the label selector come from the operator's command-line flags, and the Kubernetes calls are fixed in code. Tool-argument scoping has no surface in the scored mode. The MCP server (not scored) does expose model-callable tools, and those include reading any Secret by name.

- **Structural absence:** searched `rg -n -i 'tool_calls|toolcall|functioncall|tool_choice|tooluse|function_call'` in `pkg/ai pkg/analysis cmd/analyze` → 0 hits (No tool or function calling in any AI client or the analysis pipeline.); [pkg/ai/iai.go:71-72](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/ai/iai.go#L71-L72); [pkg/analysis/analysis.go:752-760](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analysis/analysis.go#L752-L760)

### C4 Code-execution isolation — 1.00 (high)

Nothing interprets model output or cluster content as code. The only process launch in the codebase opens a fixed OpenAI key-generation URL in the user's browser for `k8sgpt generate`. There is no shell tool, no eval, no template rendering of model text, and no plugin loading.

- **Structural absence:** searched `rg -n 'exec\.Command|os\.StartProcess|plugin\.Open'` in `pkg cmd` → 3 hits (All three hits are cmd/generate/generate.go opening the fixed https://platform.openai.com/api-keys URL in a browser; no model text reaches them.); searched `rg -n -i 'tool_calls|toolcall|functioncall|tool_choice|tooluse|function_call'` in `pkg/ai pkg/analysis cmd/analyze` → 0 hits (No tool or function calling in any AI client or the analysis pipeline.)

### C5 Untrusted input blast radius — 0.88 (high)

k8sgpt sends attacker-influenced cluster text to the model: event messages, container status messages and probe output that any workload owner can shape. The only in-prompt defence is a set of triple-dash delimiters. The pipeline is fixed in code before any of that text is read, and the model has no tools, so a hijacked model can only change the explanation the operator reads. The residual risk is the human: injected text can produce convincing but harmful remediation advice for an operator who usually holds cluster-admin, and model output is printed to the terminal without stripping control characters.

- **S L4:** Control and data flow are fixed in code: deterministic analyzers, then one text completion per result, then print; untrusted text cannot choose tools or destinations because none exist. — [pkg/analysis/analysis.go:752-760](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analysis/analysis.go#L752-L760); [pkg/ai/iai.go:71-72](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/ai/iai.go#L71-L72); [pkg/analysis/analysis.go:695-703](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analysis/analysis.go#L695-L703) (verified)
- **C L3:** Every untrusted source (events, statuses, optional pod logs, opt-in custom analyzer results) flows into the same tool-less completion. — [pkg/analyzer/pod.go:135-137](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analyzer/pod.go#L135-L137); [pkg/analyzer/pod.go:175-177](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analyzer/pod.go#L175-L177); [pkg/ai/prompts.go:4](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/ai/prompts.go#L4) (verified)
  - *To reach the next level:* Untrusted cluster text is spliced into the prompt next to the instructions (delimiters only), so third-party content is not structurally treated as data.
- **D L4:** There is no setting, config key or input that gives the model tools or actions on this path. — searched `rg -n -i 'tool_calls|toolcall|functioncall|tool_choice|tooluse|function_call'` in `pkg/ai pkg/analysis cmd/analyze` → 0 hits (No tool or function calling in any AI client or the analysis pipeline.); [pkg/ai/interactive/interactive.go:58-65](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/ai/interactive/interactive.go#L58-L65) (verified)
- **B L3:** A hijack can only shape text shown to the operator (misleading fixes, terminal escape sequences); there is no unattended egress or state change. — [pkg/analysis/output.go:103](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analysis/output.go#L103); [pkg/ai/interactive/interactive.go:65](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/ai/interactive/interactive.go#L65) (verified)
  - *To reach the next level:* The session still holds sensitive cluster data, and model output is printed raw to the terminal, so the human-mediated channel remains.
- **Cap:** none
- **Notes:** In MCP mode (not scored) the analyze tool's explanations and raw cluster objects, including Secrets, are returned to the host model, which may hold other tools. That mode is a cascading-injection and exfiltration surface outside this score.

### C6 Memory, context & configuration integrity — 0.35 (high)

Configuration is read only from user scope: the XDG config directory, an explicit --config flag, or K8SGPT_* environment variables. Nothing is loaded from the working directory. The one persistent store the model influences is the response cache: each AI answer is written verbatim and served again, without calling the model, whenever the same failure text recurs. Entries carry no provenance and never expire. The default file cache is private to the OS user (mode 0600), but the cache key ignores which cluster or context produced it. A poisoned explanation can persist across sessions, though it only affects text shown to the operator. `k8sgpt cache purge` clears it.

- **S L1:** Model responses are stored and replayed verbatim with no validation, logging or expiry; config comes only from user scope. — [pkg/analysis/analysis.go:757](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analysis/analysis.go#L757); [pkg/analysis/analysis.go:714-725](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analysis/analysis.go#L714-L725); searched `rg -n -i 'godotenv|dotenv|AGENTS\.md|CLAUDE\.md'` in `pkg cmd` → 0 hits (No working-directory .env or instruction files are loaded.) (verified)
  - *To reach the next level:* Cache writes are neither validated nor provenance-tagged nor time-limited.
- **C L1:** The config path is limited to user scope; the response cache is uncontrolled. — [cmd/root.go:93-102](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/cmd/root.go#L93-L102); [pkg/util/util.go:181-186](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/util/util.go#L181-L186) (verified)
  - *To reach the next level:* The response cache, the only model-influenced store, has no write control.
- **D L2:** The default file cache lives in the user's XDG cache directory with 0600 files; the model cannot choose keys. — [pkg/cache/file_based.go:100-107](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/cache/file_based.go#L100-L107) (verified)
  - *To reach the next level:* Cache keys are not scoped to cluster/context, and remote caches (S3/GCS/Azure) can be shared across users once configured.
- **B L2:** A poisoned answer persists across the user's sessions with no TTL but only influences printed text. — [pkg/analysis/analysis.go:714-725](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analysis/analysis.go#L714-L725); [pkg/cache/file_based.go:26-47](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/cache/file_based.go#L26-L47) (verified)
  - *To reach the next level:* Cache entries are not easily inspected (listing shows only hashed keys) and there is no review or rollback.
- **Cap:** none

### C7 Third-party extensions — 0.38 (high)

Nothing third-party is enabled by default. There are two opt-in extension paths. The first is custom analyzers: remote gRPC services the operator registers and enables with the -z flag. They receive an empty request and no credentials, but the connection is unauthenticated plaintext and nothing pins or verifies the endpoint. The second is `integration activate keda`, which installs a version-pinned KEDA Helm chart from the official repository using the operator's kubeconfig, with no digest or signature check. The repository and version can be overridden through environment variables. A malicious chart would run in the cluster with whatever RBAC it declares.

- **S L2:** The KEDA chart version is pinned (2.11.2) to the official chart repository; custom analyzers are network data sources, not code k8sgpt runs. — [pkg/integration/keda/keda.go:22-23](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/integration/keda/keda.go#L22-L23) (verified)
  - *To reach the next level:* No chart digest or provenance verification, and custom analyzer endpoints have no identity check or TLS.
- **C L1:** Only the KEDA install is pinned; custom analyzers are reached over insecure gRPC with no verification. — [pkg/custom/client.go:21](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/custom/client.go#L21) (verified)
  - *To reach the next level:* Custom analyzer endpoints are not verified (no TLS, no pinning of the service).
- **D L2:** Custom analyzers require `custom-analyzer add` plus the -z flag; KEDA requires an explicit `integration activate keda`; neither shows the exact chart, image or permissions. — [cmd/analyze/analyze.go:221](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/cmd/analyze/analyze.go#L221); [pkg/integration/integration.go:93-95](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/integration/integration.go#L93-L95) (verified)
  - *To reach the next level:* The install step does not show the repository, version and RBAC that will be applied.
- **B L1:** The KEDA chart is installed with the operator's kubeconfig and runs cluster-wide with its own declared RBAC; custom analyzers get no credentials (empty RunRequest). — [pkg/integration/keda/keda.go:62-76](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/integration/keda/keda.go#L62-L76); [pkg/custom/client.go:35](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/custom/client.go#L35) (verified)
  - *To reach the next level:* Installed charts are not confined to scoped credentials or limited RBAC.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.38 (high)

The AI provider key is written in plaintext to the user's k8sgpt.yaml through viper. No file mode is set, so viper's default world-readable 0644 likely applies (inferred from the library default, not verified). `k8sgpt dump` masks the key and `auth list` never prints it. There is no telemetry or crash reporting, verbose output prints only the base URL and model, and cached answers are stored 0600. Cluster object names and event text go to the provider unmasked unless the operator passes --anonymize, which is off by default and masks only identifiers.

- **S L1:** The key comes from a prompt or flag and is stored in plaintext YAML; the only masking is in `dump` and the opt-in --anonymize for model-bound identifiers. — [cmd/auth/add.go:154-158](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/cmd/auth/add.go#L154-L158); [cmd/dump/dump.go:62-68](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/cmd/dump/dump.go#L62-L68) (verified)
  - *To reach the next level:* No keychain/secret store or explicit restrictive permissions on the stored key.
- **C L2:** Logs (verbose), dumps, and stored transcripts do not carry the key; model-bound messages carry cluster identifiers unless --anonymize is set. — [pkg/analysis/analysis.go:204-207](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analysis/analysis.go#L204-L207); [pkg/analysis/analysis.go:667-671](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analysis/analysis.go#L667-L671) (verified)
  - *To reach the next level:* Model-bound messages are not redacted by default, and anonymization does not cover free-text secrets in event or log lines.
- **D L2:** No telemetry exists and verbose logging is off by default; model-bound anonymization is opt-in. — [cmd/analyze/analyze.go:203](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/cmd/analyze/analyze.go#L203); [cmd/root.go:88](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/cmd/root.go#L88) (verified)
  - *To reach the next level:* Anonymization is off by default and the config file holding the key is not written with restrictive permissions.
- **B L1:** A leaked provider key is long-lived and as broad as the operator made it; the model never sees it. — [pkg/ai/openai.go:47-48](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/ai/openai.go#L47-L48) (verified)
  - *To reach the next level:* Keys are long-lived and not scoped or rotated by the tool.
- **Cap:** none
- **Notes:** File permission of the config file is inferred from spf13/viper v1.19.0's default config permissions (0644); k8sgpt never calls SetConfigPermissions.

### C9 Audit & traceability — 0.00 (high)

The CLI keeps no record of what it did: which resources it read, what text it sent to the AI provider, or what came back. The only output is the report on stdout, plus cached answers stored under hashed keys. Debug lines appear only with --verbose and go to stdout. The serve mode, which is not scored, does log each gRPC request with structured zap fields.

- **S L0:** No record of analyzer reads or provider calls exists; debug prints only with --verbose. — searched `rg -n 'zap\.|log\.Print|logger\.' --glob '!*_test.go'` in `pkg/analysis pkg/ai cmd/analyze` → 0 hits (Nothing on the CLI analyze path writes a log record.); [cmd/analyze/analyze.go:85-88](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/cmd/analyze/analyze.go#L85-L88) (verified)
  - *To reach the next level:* No structured record of each provider call (prompt, provider, result, timestamp).
- **C L0:** Nothing on the analyze path is recorded. — searched `rg -n 'zap\.|log\.Print|logger\.' --glob '!*_test.go'` in `pkg/analysis pkg/ai cmd/analyze` → 0 hits (Nothing on the CLI analyze path writes a log record.) (verified)
  - *To reach the next level:* No path is recorded.
- **D L0:** There is no audit record to turn on. — searched `rg -n 'zap\.|log\.Print|logger\.' --glob '!*_test.go'` in `pkg/analysis pkg/ai cmd/analyze` → 0 hits (Nothing on the CLI analyze path writes a log record.) (verified)
  - *To reach the next level:* No default-on audit record.
- **B L0:** With no records, nothing survives a crash or proves what was sent. — searched `rg -n 'zap\.|log\.Print|logger\.' --glob '!*_test.go'` in `pkg/analysis pkg/ai cmd/analyze` → 0 hits (Nothing on the CLI analyze path writes a log record.) (verified)
  - *To reach the next level:* Records are not written or flushed at all.
- **Cap:** none

### C10 Limits & kill switch — 0.50 (high)

There is no agent loop to run away. Each run makes one completion per detected problem, each OpenAI completion is capped at 2,048 tokens, and Kubernetes concurrency is capped at 100. No limit applies to the number of calls per run or their total cost, and AI and Kubernetes requests use a background context with no timeout. Ctrl-C ends the process, which stops in-flight requests, and nothing keeps running in the background.

- **S L2:** The call count is fixed by the deterministic analysis (not the model) and each completion is token-capped in code; Ctrl-C terminates the process. — [pkg/ai/openai.go:41](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/ai/openai.go#L41); [pkg/ai/openai.go:100](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/ai/openai.go#L100); [pkg/analysis/analysis.go:646](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analysis/analysis.go#L646) (verified)
  - *To reach the next level:* No wall-clock limit or overall cost cap per run.
- **C L2:** There are no sub-agents, background tasks or spawned processes to escape the run's bounds. — [pkg/analysis/analysis.go:130-131](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analysis/analysis.go#L130-L131) (verified)
  - *To reach the next level:* No per-request timeouts on AI or Kubernetes calls (context.Background()).
- **D L2:** Defaults are hard-coded (2048 tokens, concurrency 10 capped at 100) and the model cannot change them. — [pkg/analysis/analysis.go:403-410](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/pkg/analysis/analysis.go#L403-L410) (verified)
  - *To reach the next level:* No hard ceiling on the number of provider calls per run.
- **B L2:** Spend scales with the number of failing resources; stopping kills the whole process. — [cmd/analyze/analyze.go:136-141](https://github.com/k8sgpt-ai/k8sgpt/blob/635d9b122318a41ada1bfcf9153801365fdcaf77/cmd/analyze/analyze.go#L136-L141) (verified)
  - *To reach the next level:* No tight per-run time or cost ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: workload-controlled event, status and probe messages enter the prompt (pkg/analyzer/pod.go:135-177) · [B] sensitive data/systems: cluster-wide reads with the operator's kubeconfig (pkg/analysis/analysis.go:97-100) · [C] state change / egress: none reachable by the model; its answer is only cached and printed (pkg/analysis/analysis.go:752-760, pkg/analysis/output.go:103) · Same default session? No

## Highest-impact improvements
1. Write a structured local record of every run (resources analyzed, provider, prompt hash or text, response, timestamp) outside the working directory. — C9 S L0→L2, +0.150 before caps (Playbook 1, step 3)
2. Default to a read-only identity: impersonate a view-only ServiceAccount (or ship a kubeconfig generator for one) unless the operator opts out. — C1 D L0→L3, +0.150 before caps (Playbook 4, step 1)
3. Write the config file with 0600 permissions (viper.SetConfigPermissions) or store provider keys in the OS keychain. — C8 S L1→L2, +0.075 before caps (Playbook 4, step 2)
4. Give cache entries a TTL and scope cache keys by cluster/context and model, so a poisoned answer cannot persist indefinitely or bleed across clusters. — C6 S L1→L2, +0.075 before caps (Playbook 2, step 1)
5. Require TLS and an expected server identity for custom analyzers, and verify the KEDA chart digest before install. — C7 C L1→L2, +0.075 before caps (Playbook 3, step 2)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was built, run or network-probed.
- Only the CLI analyze --explain flow was scored. `k8sgpt serve` and `serve --mcp` (tools that list/get Secrets, a config tool that persists custom-analyzer URLs and remote cache endpoints) would score much lower and were not rated.
- The k8sgpt-operator (separate repository) and its result sinks were not examined; the Helm chart in this repo grants get/list/watch on all resources (charts/k8sgpt/templates/role.yaml).
- Config-file permissions (C8) are inferred from spf13/viper's default, not from k8sgpt code.
- Individual AI backend clients other than OpenAI were not each reviewed for token caps and timeouts.
- No reviewer-directed instructions were found in the repository (searched README, docs and source for reviewer/auditor/ignore-instructions text).
