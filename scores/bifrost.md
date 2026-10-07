# Defense-in-Depth Score: Bifrost

**Repo:** https://github.com/maximhq/bifrost · **Commit:** `3b31be0037e1414edc4644edcba3d33011518a47` · **Reviewed:** 2026-10-04
**What it is:** High-performance AI gateway unifying 20+ LLM providers behind an OpenAI-compatible API, with an MCP gateway and agent mode for tool calling.
**Category:** Agent Frameworks
**Scored configuration:** Bifrost HTTP gateway as shipped (npx or docker run, no config.json), with MCP clients added using default fields (no auto-execute, shared credentials, no virtual keys).
**Agent surface (default):** code execution opt-in · filesystem write no · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents no · external communication opt-in

## Score: 4.5 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L3 | L2 | L0 | L2 | 0.47 | G1 | **0.47** (alt) | High |
| C2 | Approval gates | L2 | L3 | L2 | L1 | 0.53 | — | **0.53** | High |
| C3 | Tool & action scoping | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C4 | Code-execution isolation | L3 | L3 | L3 | L1 | 0.65 | — | **0.65** | High |
| C5 | Untrusted input blast radius | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C6 | Memory, context & configuration integrity | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | Medium |
| C7 | Third-party extensions | L1 | L0 | L2 | L0 | 0.17 | — | **0.17** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |


Bifrost's MCP agent mode is cautious where it counts: no tool runs unattended unless the operator lists it, and model-written code-mode scripts run in a capability-limited Starlark interpreter. The dominant risk is the default deployment posture: admin and inference auth are off, the Docker image listens on all interfaces, and every caller shares the gateway's provider keys and MCP credentials. Native plugins load in-process without integrity checks, and secrets are stored in plaintext unless an encryption key is set.

## Critical gaps
- Native .so plugins are dlopen'd into the gateway process (optionally downloaded from a URL) with no hash or signature check, and stdio MCP servers inherit the gateway environment, so a malicious extension gets every provider key. (ASI04, T17, LLM03; C7) — [framework/plugins/soloader.go:30](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/framework/plugins/soloader.go#L30); [framework/plugins/soloader.go:42](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/framework/plugins/soloader.go#L42); [core/mcp/clientmanager.go:3329](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/clientmanager.go#L3329)

## Criterion details

### C1 Identity & least privilege — 0.47 (high)

Bifrost holds the operator's provider API keys and each MCP server's credential, and by default every caller shares them. Out of the box neither the management API nor the inference API requires authentication, and a request with no virtual key gets no per-caller tool restriction at all. Virtual keys with per-key MCP tool grants, per-user OAuth, per-user headers and RFC 8693 token exchange exist and are good designs, but all are opt-in. Several dangerous registrations (stdio servers, native plugins, env/vault secret references, private-network MCP targets) are refused while auth is off, which limits what an unauthenticated caller can widen.

- **default configuration** (default; raw 0.33 → 0.33)
  - **S L2:** Default MCP clients use one static, operator-configured credential per server (none/shared headers/shared OAuth) shared by all callers. — [core/schemas/mcp.go:547](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/schemas/mcp.go#L547); [core/schemas/mcp.go:591](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/schemas/mcp.go#L591) (verified)
    - *To reach the next level:* No per-tool or per-request credential narrowing in the default auth types; per-user and token-exchange modes are opt-in.
  - **C L1:** Callers without a virtual key skip the governance access resolution, so no per-principal authorization applies to their tool calls; stdio MCP servers are launched through mcp-go, which (library behaviour, inferred) appends the gateway's full environment. — [plugins/governance/main.go:1090](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/plugins/governance/main.go#L1090); [core/mcp/clientmanager.go:3329](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/clientmanager.go#L3329) (verified)
    - *To reach the next level:* Every tool path, including stdio subprocesses, should pass a per-principal check and receive a scrubbed environment.
  - **D L1:** Dashboard/admin auth and inference auth are off by default and the Docker image binds 0.0.0.0, so any network caller acts as local admin; stdio, .so plugin, secret-reference and private-target registrations are refused in that mode. — [transports/bifrost-http/handlers/middlewares.go:1517](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/bifrost-http/handlers/middlewares.go#L1517); [transports/bifrost-http/lib/config.go:689](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/bifrost-http/lib/config.go#L689); [transports/Dockerfile:70](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/Dockerfile#L70); [transports/bifrost-http/handlers/mcp.go:2008](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/bifrost-http/handlers/mcp.go#L2008) (verified)
    - *To reach the next level:* Default should require authenticated admin and inference access; tool allowlists and provider keys are editable by any unauthenticated caller.
  - **B L1:** A hijacked or unauthenticated caller can drive every provider key and every shared MCP credential the gateway holds, i.e. write access across multiple external systems. — [plugins/governance/main.go:1090](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/plugins/governance/main.go#L1090); [core/schemas/mcp.go:591](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/schemas/mcp.go#L591) (verified)
    - *To reach the next level:* Per-tenant scoping with short-lived credentials would bound a compromise to one tenant.
- **opt-in per-user OAuth / per-user headers / token exchange with virtual-key MCP grants** (alt; raw 0.47, cap G1 → 0.47) ← counted
  - **S L3:** Per-user MCP auth types resolve credentials per caller at call time and token exchange downscopes to a configured audience. — [core/schemas/mcp.go:547](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/schemas/mcp.go#L547); [core/schemas/mcp.go:544](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/schemas/mcp.go#L544) (verified)
    - *To reach the next level:* Credentials are not issued per task and revoked after use.
  - **C L2:** Applies to MCP clients configured with those auth types; provider keys and other clients remain shared. — [core/schemas/mcp.go:544](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/schemas/mcp.go#L544) (verified)
    - *To reach the next level:* All tool paths would need per-principal credentials, including provider keys and stdio servers.
  - **D L0:** Off by default; requires the operator to configure per-user auth types and virtual keys. — [transports/bifrost-http/lib/config.go:689](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/bifrost-http/lib/config.go#L689) (verified)
    - *To reach the next level:* Should be the default MCP auth posture.
  - **B L2:** With per-user credentials a hijack is limited to the requesting user's grants on each server. — [core/schemas/mcp.go:544](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/schemas/mcp.go#L544) (verified)
    - *To reach the next level:* Credentials are long-lived OAuth tokens rather than minutes-lived task tokens.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C2 Approval gates — 0.53 (high)

Bifrost never auto-runs an MCP tool unless the operator lists it in tools_to_auto_execute; that list is empty by default, unknown tools are treated as not auto-executable, and the same rule is re-checked inside code-mode scripts at the exact point each nested tool is called. Anything not auto-approved is returned verbatim to the calling application, which must execute it explicitly. Bifrost does not itself provide a human approval step: whether a person sees the call is up to the application, and there is no argument-level policy. With dashboard auth off by default, any caller who can reach the management API can set the auto-execute list to '*', and there is no undo for tool actions.

- **S L2:** Per-tool risk tiering (auto vs manual) is enforced in code and non-auto calls are handed back to the caller exactly as the model produced them, but human approval is delegated to the application. — [core/mcp/agent.go:267](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/agent.go#L267); [core/mcp/utils.go:876](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/utils.go#L876) (verified)
  - *To reach the next level:* No Bifrost-enforced human approval step or argument-level allow/deny rules.
- **C L3:** Agent loop, unknown tools and every nested code-mode call traverse the same auto-execute check (unattended flag set on every agent-loop tool context). — [core/mcp/agent.go:261](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/agent.go#L261); [core/mcp/agent.go:300](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/agent.go#L300); [core/mcp/codemode/starlark/executecode.go:436](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/codemode/starlark/executecode.go#L436) (verified)
  - *To reach the next level:* Compound/argument parsing and default rejection of unknown tools at the execute endpoint are not present; arguments are not policy-checked.
- **D L2:** Deny-by-default for auto-execution, but the auto-execute list (including '*') can be changed through the management API, which is unauthenticated by default. — [core/mcp/utils.go:876](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/utils.go#L876); [transports/bifrost-http/handlers/middlewares.go:1517](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/bifrost-http/handlers/middlewares.go#L1517) (verified)
  - *To reach the next level:* Changing approval posture should require authenticated operator action with a loud flag.
- **B L1:** Auto-executed or application-executed MCP tools can take arbitrary external actions with no checkpoint, rollback or dry-run in Bifrost; VK rate limits/budgets are opt-in. — searched `rg -n -i 'rollback|undo|dry.?run' -g '!*_test.go'` in `core/mcp` → 2 hits (Both hits are comments about client-config update internals; no rollback, undo or dry-run exists for tool actions.) (verified)
  - *To reach the next level:* No previews, dry-runs or rollback for external actions.
- **Cap:** none

### C3 Tool & action scoping — 0.38 (high)

Scoping is done at the tool-name level: each MCP client exposes only the tools in tools_to_execute (deny-by-default), per-virtual-key grants and request headers can only narrow that set, and the execute path re-applies the same filters so a hidden tool can't be called by name. Tool arguments are passed to the MCP server without validation against bounds or allowlists. HTTP MCP targets registered without admin auth are pinned to public addresses on every dial.

- **S L1:** Name-level allowlists and SSRF pinning of MCP targets exist, but tool arguments are passed through unvalidated. — [core/mcp/exec.go:165](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/exec.go#L165); [transports/bifrost-http/handlers/mcp.go:2022](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/bifrost-http/handlers/mcp.go#L2022) (verified)
  - *To reach the next level:* Arguments should be validated in code against allowlists and bounds (paths, hosts, quantities).
- **C L2:** The same allowlist and request-narrowing filters apply on discovery and execution for every MCP client and code-mode call. — [core/mcp/exec.go:165](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/exec.go#L165); [core/mcp/codemode/starlark/executecode.go:436](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/codemode/starlark/executecode.go#L436) (verified)
  - *To reach the next level:* No shared argument-validation layer that every tool inherits.
- **D L2:** tools_to_execute nil or empty means no tools; the operator must explicitly enable each tool or '*'. — [core/schemas/mcp.go:554](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/schemas/mcp.go#L554); [core/mcp/utils.go:844](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/utils.go#L844) (verified)
  - *To reach the next level:* Per-task tool allowlists by default; limited by weak argument validation (C/D at most one level above S).
- **B L1:** An enabled MCP tool does whatever its upstream server allows with the shared credential; Bifrost adds no quantity bounds. — [plugins/governance/main.go:1090](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/plugins/governance/main.go#L1090) (verified)
  - *To reach the next level:* Quantity bounds (max rows, recipients, amounts) on consequential tool calls.
- **Cap:** none

### C4 Code-execution isolation — 0.65 (high)

The only place model-written code runs is opt-in code mode, which executes scripts in an embedded Starlark interpreter (a memory-safe Go runtime) whose only capabilities are the MCP tool functions Bifrost injects; the package imports no OS, process or network modules, and scripts are cancelled when the execution timeout fires. There is no unsandboxed fallback. The interpreter runs inside the gateway process, so a hypothetical escape would land in a process that holds every provider key and full network access. Stdio MCP servers run as host subprocesses but their commands are operator-configured, not model-generated.

- **S L3:** Embedded Starlark interpreter with only injected host functions; no os/exec/http imports. — [core/mcp/codemode/starlark/executecode.go:230](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/codemode/starlark/executecode.go#L230); searched `rg -n '"os"|"os/exec"|"net/http"' -g '!*_test.go'` in `core/mcp/codemode/starlark/` → 0 hits (The Starlark code-mode package imports no OS, process or HTTP package; scripts only see injected tool builtins.) (verified)
  - *To reach the next level:* Kernel-level isolation (microVM, gVisor, WASM capability imports) would be L4.
- **C L3:** Every model-reachable code path (executeToolCode) runs in Starlark; operator-configured stdio servers run on the host. — [core/mcp/codemode/starlark/executecode.go:436](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/codemode/starlark/executecode.go#L436); [core/mcp/clientmanager.go:3329](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/clientmanager.go#L3329) (verified)
  - *To reach the next level:* Processes spawned by extensions (stdio MCP servers) are not isolated.
- **D L3:** Whenever code mode is used it is always Starlark; there is no switch or fallback to host execution. — [core/mcp/codemode/starlark/executecode.go:230](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/codemode/starlark/executecode.go#L230); [core/mcp/codemode/starlark/executecode.go:345](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/codemode/starlark/executecode.go#L345) (verified)
  - *To reach the next level:* Isolation policy outside anything an unauthenticated management caller can change is not demonstrated for code-mode enablement.
- **B L1:** An escape lands inside the gateway process, which holds all provider keys and unrestricted egress (container runs as non-root UID 1000). — [transports/Dockerfile:106](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/Dockerfile#L106); [core/mcp/toolmanager.go:778](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/toolmanager.go#L778) (verified)
  - *To reach the next level:* Run code mode out of process with no credentials and restricted egress.
- **Cap:** none

### C5 Untrusted input blast radius — 0.45 (high)

MCP tool results and upstream servers' instructions go straight into the model's context (server instructions as a system message), with no provenance tagging or taint tracking. What limits a hijacked session is the auto-execute allowlist: the agent loop only runs operator-listed tools unattended and hands everything else back to the application. That holds regardless of what content was read, so it does not stop an auto-listed tool from being used for exfiltration. Shared MCP credentials mean a hijack in one user's session can reach data visible to the shared identity.

- **S L2:** Non-allowlisted tools never run unattended, even after untrusted content is read; allowlisted ones (including egress-capable) do. — [core/mcp/agent.go:267](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/agent.go#L267); [core/mcp/utils.go:876](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/utils.go#L876) (verified)
  - *To reach the next level:* No taint-based rule-of-two enforcement or quarantined-model design.
- **C L2:** The rule covers tool results and nested code-mode calls, but upstream server instructions are injected with system-role standing. — [core/mcp/toolmanager.go:616](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/toolmanager.go#L616); [core/mcp/codemode/starlark/executecode.go:436](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/codemode/starlark/executecode.go#L436) (verified)
  - *To reach the next level:* Tool descriptions and server instructions should be treated as data, not system messages.
- **D L2:** On by default via the empty auto-execute list; the operator (or any unauthenticated management caller) can widen it silently. — [core/mcp/utils.go:876](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/utils.go#L876); [transports/bifrost-http/handlers/middlewares.go:1517](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/bifrost-http/handlers/middlewares.go#L1517) (verified)
  - *To reach the next level:* Widening should be explicit, authenticated and warned.
- **B L1:** By default nothing runs unattended, but operators commonly auto-list tools and the shared credential spans users, so a hijack can reach other users' data (one level lower for multi-tenant). — [core/mcp/agent.go:267](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/agent.go#L267); [plugins/governance/main.go:1090](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/plugins/governance/main.go#L1090) (verified)
  - *To reach the next level:* Per-user credentials plus approval on egress-capable tools would contain cross-user reach.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.50 (medium)

The model has no memory tool and Bifrost auto-loads no workspace files: configuration comes from the operator's app directory or database, and there is no .env loading. The persistence the model can influence is the opt-in semantic cache, which stores responses (including tool calls) and serves them to later requests sharing a caller-supplied cache key until the TTL expires. Cached entries are not validated or provenance-tagged.

- **S L2:** No model-writable memory and no repo-controlled config; cached responses are stored without validation. — searched `rg -n 'godotenv|dotenv' --type go` in `core framework transports plugins` → 0 hits (No .env auto-loading from the working directory.); [plugins/semanticcache/main.go:248](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/plugins/semanticcache/main.go#L248) (verified)
  - *To reach the next level:* Cache writes are not gated or validated, and no versioned rollback exists.
- **C L2:** Config loading is operator-scoped; the semantic cache store is uncontrolled. — [plugins/semanticcache/main.go:248](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/plugins/semanticcache/main.go#L248) (verified)
  - *To reach the next level:* Cached entries should carry provenance end to end.
- **D L2:** Semantic cache entries are bucketed by a caller-supplied cache key (inferred to be enforced in vector queries). — [plugins/semanticcache/main.go:248](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/plugins/semanticcache/main.go#L248) (inferred)
  - *To reach the next level:* Isolation keyed on a caller-supplied value rather than the authenticated principal.
- **B L2:** A poisoned cache entry persists until TTL and only influences text or tool calls returned to the caller. — [plugins/semanticcache/main.go:248](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/plugins/semanticcache/main.go#L248) (verified)
  - *To reach the next level:* Session-scoped or easily purged per-entry cache with review.
- **Cap:** none

### C7 Third-party extensions — 0.17 (high)

Two kinds of third-party code can run: native Go .so plugins, which are downloaded from a URL or read from disk and dlopen'd into the gateway process with no hash or signature check, and stdio MCP servers, which run whatever command the operator configures (often an unpinned npx package) as a subprocess that inherits the gateway environment. Neither is enabled by default, and both are refused while dashboard auth is off, so only an authenticated admin (or config.json) can add them. Once added, a malicious plugin has everything the gateway has.

- **S L1:** Operator-chosen sources with no pinning or integrity verification; .so plugins can be fetched from any http(s) URL. — [framework/plugins/soloader.go:30](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/framework/plugins/soloader.go#L30); [framework/plugins/soloader.go:42](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/framework/plugins/soloader.go#L42); searched `rg -n -i 'sha256|checksum|signature' -g '!*_test.go'` in `framework/plugins` → 18 hits (All 18 hits are Go type-signature casts/comments; no hash or signature check on downloaded or local .so plugins.) (verified)
  - *To reach the next level:* Pin versions and verify hashes/signatures.
- **C L0:** Neither .so plugins nor stdio MCP servers are verified. — searched `rg -n -i 'sha256|checksum|signature' -g '!*_test.go'` in `framework/plugins` → 18 hits (All 18 hits are Go type-signature casts/comments; no hash or signature check on downloaded or local .so plugins.); [core/mcp/clientmanager.go:3329](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/clientmanager.go#L3329) (verified)
  - *To reach the next level:* Verify at least one extension type.
- **D L2:** Nothing third-party enabled by default; adding .so plugins or stdio servers requires genuine admin auth (refused when auth is bypassed). — [transports/bifrost-http/handlers/plugins.go:327](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/bifrost-http/handlers/plugins.go#L327); [transports/bifrost-http/handlers/mcp.go:2008](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/bifrost-http/handlers/mcp.go#L2008) (verified)
  - *To reach the next level:* Consent screen showing exact package, version and permissions (also limited by C/D at most one above S).
- **B L0:** .so plugins run in-process with all provider keys; stdio servers inherit the full gateway environment (mcp-go library behaviour). — [framework/plugins/soloader.go:42](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/framework/plugins/soloader.go#L42); [core/mcp/clientmanager.go:3329](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/clientmanager.go#L3329) (verified)
  - *To reach the next level:* Separate process with scrubbed environment for every extension.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.40 (high)

Secrets are typed (SecretVar) and redacted when configuration is read back through the API, can reference env or vault values, and internal errors are sanitized before reaching clients. Encryption at rest is only on when an encryption key is set; otherwise provider keys and MCP credentials sit in plaintext in the local database. Prompt and response content logging is on by default. Nothing is sent to a vendor telemetry service, and secrets are never placed in model context, but stdio MCP subprocesses inherit the gateway's environment.

- **S L2:** Type-level masking on API reads; AES-GCM encryption exists but returns plaintext when no key is configured. — [core/schemas/secretvar.go:273](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/schemas/secretvar.go#L273); [framework/encrypt/encrypt.go:74](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/framework/encrypt/encrypt.go#L74); [framework/encrypt/encrypt.go:33](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/framework/encrypt/encrypt.go#L33) (verified)
  - *To reach the next level:* Encryption at rest by default or a secret manager, plus redaction on all major paths.
- **C L2:** API responses and client errors are protected; subprocess environments and content logs are not. — [core/schemas/secretvar.go:273](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/schemas/secretvar.go#L273); [core/mcp/clientmanager.go:3329](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/clientmanager.go#L3329) (verified)
  - *To reach the next level:* Scrub subprocess environments and redact secrets in logs.
- **D L1:** No third-party telemetry, but full content logging is on by default and stored unredacted in logs.db. — [transports/bifrost-http/lib/config.go:686](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/bifrost-http/lib/config.go#L686); [transports/bifrost-http/lib/config.go:687](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/bifrost-http/lib/config.go#L687) (verified)
  - *To reach the next level:* Content logging should be opt-in or redacted by default.
- **B L1:** Long-lived provider API keys held by the process and inherited by stdio subprocesses. — [core/mcp/clientmanager.go:3329](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/clientmanager.go#L3329); [framework/encrypt/encrypt.go:33](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/framework/encrypt/encrypt.go#L33) (verified)
  - *To reach the next level:* Short-lived, scoped, rotatable credentials.
- **Cap:** none

### C9 Audit & traceability — 0.45 (high)

Logging is on by default and records every MCP tool execution through the plugin hooks, with the virtual key and user that made the call and a link back to the originating agent request. Records go to a local SQLite (or Postgres) log store via a batched asynchronous writer, outside any workspace. There is no approval record (approval lives in the calling application), no tamper evidence, and with dashboard auth off by default anyone who can reach the API can delete MCP logs.

- **S L2:** Structured MCP tool log entries with arguments, status and governance (VK/user) attribution. — [plugins/logging/main.go:2864](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/plugins/logging/main.go#L2864); [plugins/logging/main.go:281](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/plugins/logging/main.go#L281) (verified)
  - *To reach the next level:* No approver attribution or tamper-evident storage.
- **C L2:** All MCP tool executions (agent loop, execute endpoint, code-mode nested calls) pass the plugin hooks. — [plugins/logging/main.go:2864](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/plugins/logging/main.go#L2864); [core/mcp/exec.go:165](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/exec.go#L165) (verified)
  - *To reach the next level:* Approvals/denials, configuration changes and credential use are not recorded as audit events.
- **D L2:** On by default and outside the workspace, but DELETE /api/mcp-logs is reachable without auth in the default posture. — [transports/bifrost-http/lib/config.go:686](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/bifrost-http/lib/config.go#L686); [transports/bifrost-http/handlers/logging.go:422](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/bifrost-http/handlers/logging.go#L422) (verified)
  - *To reach the next level:* Log deletion should require authenticated operator action and itself be logged.
- **B L1:** Writes are batched asynchronously with retries; records in the queue are lost on crash and actions do not wait for the record. — [plugins/logging/writer.go:32](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/plugins/logging/writer.go#L32) (verified)
  - *To reach the next level:* Flush durably per action.
- **Cap:** none

### C10 Limits & kill switch — 0.50 (high)

The agent loop stops after 10 iterations by default, each MCP tool call has a 30-second timeout, and code-mode scripts are cancelled when their timeout fires. Token and cost budgets and rate limits exist through virtual keys but are opt-in. There is no wall-clock cap on the whole agent run, and cancelling a timeout stops waiting but cannot recall an external action already sent.

- **S L2:** Iteration cap plus per-tool timeout enforced in code; Starlark thread cancelled on timeout. — [core/mcp/agent.go:168](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/agent.go#L168); [core/mcp/toolmanager.go:778](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/toolmanager.go#L778); [core/mcp/codemode/starlark/executecode.go:345](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/codemode/starlark/executecode.go#L345) (verified)
  - *To reach the next level:* No default token/cost cap or session wall-clock limit.
- **C L2:** Top-level agent loop plus tool timeouts, including each nested code-mode tool call. — [core/mcp/agent.go:168](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/agent.go#L168); [core/mcp/toolmanager.go:778](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/core/mcp/toolmanager.go#L778) (verified)
  - *To reach the next level:* Nested calls do not count against a shared run budget.
- **D L2:** Sensible defaults (depth 10, 30 s) that the operator can change; the model cannot raise them. — [transports/bifrost-http/lib/config.go:694](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/bifrost-http/lib/config.go#L694); [transports/bifrost-http/lib/config.go:695](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/bifrost-http/lib/config.go#L695) (verified)
  - *To reach the next level:* Hard ceilings that configuration cannot exceed.
- **B L2:** Moderate ceilings; in-flight upstream tool calls complete on the remote server. — [transports/bifrost-http/lib/config.go:694](https://github.com/maximhq/bifrost/blob/3b31be0037e1414edc4644edcba3d33011518a47/transports/bifrost-http/lib/config.go#L694) (verified)
  - *To reach the next level:* Tight per-run time and cost ceilings by default.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: MCP tool results and server instructions enter model context (core/mcp/toolmanager.go:616) · [B] sensitive data/systems: Shared provider keys and MCP credentials held by the gateway (plugins/governance/main.go:1090) · [C] state change / egress: Operator-auto-listed MCP tools execute unattended (core/mcp/agent.go:267) · Same default session? Yes

## Highest-impact improvements
1. Require authenticated admin access to the management API by default (or bind the Docker image to localhost until auth is configured). — C1 D L1→L2, +0.050 before caps (Playbook 4)
2. Pass stdio MCP servers an explicit, scrubbed environment instead of inheriting the gateway's. — C7 B L0→L2, +0.100 before caps (Playbook 3)
3. Require a pinned SHA-256 for every .so plugin before dlopen. — C7 S L1→L3, +0.150 before caps (Playbook 3)
4. Default content logging to off or redacted. — C8 D L1→L2, +0.050 before caps (Playbook 4)
5. Write audit records synchronously for MCP tool executions and block log deletion without auth. — C9 B L1→L2, +0.050 before caps (Playbook 1 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Stdio subprocess environment inheritance is inferred from mark3labs/mcp-go v0.43.2 behaviour (NewStdio appends os.Environ()); the library source was not in the clone.
- Enterprise-only features (guardrails, clustering, RBAC beyond the OSS code), the UI, the CLI, terraform and helm charts were not examined in depth.
- Semantic-cache namespace enforcement in vector queries was inferred, not traced end to end.
- No reviewer-steering text was found in AGENTS.md, README.md or SECURITY.md.
