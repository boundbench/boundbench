# Defense-in-Depth Score: MS 365 MCP Server

**Repo:** https://github.com/Softeria/ms-365-mcp-server · **Commit:** `c47197109dcf8ea6cea91ee08f7417adfbeac72b` (v0.158.0) · **Reviewed:** 2026-10-03
**What it is:** MCP server for Microsoft 365 (Outlook, Calendar, OneDrive, Teams) via Graph API
**Category:** AI Assistants
**Scored configuration:** stdio transport launched as `npx -y @softeria/ms-365-mcp-server` with no flags: personal mode, device-code MSAL login, default app registration, all personal-mode tools including writes.
**Agent surface (default):** code execution no · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 4.9 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L1 | L0 | 0.35 | — | **0.35** | High |
| C2 | Approval gates | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L2 | L2 | L3 | L0 | 0.45 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L1 | L2 | SA | L2 | 0.53 | C6-REPOCONFIG | **0.25** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C9 | Audit & traceability | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |

Controls where a risk surface exists: 2.98 / 8.0 (37%); 2 criteria scored SA (surface absent).

As shipped, this server gives the model one delegated token that can read, write and send across the user's mail, OneDrive, calendar, contacts and notes, and it applies no approval of its own. The biggest risk is prompt injection through inbound email: one malicious message can lead the model to send private data out by email or delete items, and the only checks are the host client's prompts. Engineering hygiene is strong (encrypted token cache, log redaction, a structured audit log, a .env allowlist), but one tool's risk annotation is inaccurate and read-only mode does not cover every path. For least privilege, run with --read-only or a preset.

## Critical gaps
- Default stdio mode holds one delegated token with write access across the user's mail (including send), OneDrive, calendar, contacts, tasks and notes; a hijacked model inherits the whole account. (ASI03, T3; C1) — [src/endpoints.json:80-85](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L80-L85); [src/auth.ts:437-476](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/auth.ts#L437-L476)
- Default session reads attacker-controllable email and files, holds private data, and can send mail or delete items with no server-side gate (Rule of Two violated). (ASI01, T6, LLM01; C5) — [src/endpoints.json:12](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L12); [src/endpoints.json:83](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L83); [src/graph-tools.ts:487-489](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L487-L489)
- A .env in the working directory silently sets the OAuth client ID, client secret, tenant and cloud when the operator has not, so a cloned repo can redirect which app registration and tenant the user authenticates to. (ASI06, T1; C6) — [src/load-env.ts:23-28](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/load-env.ts#L23-L28); [src/load-env.ts:43](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/load-env.ts#L43); [src/secrets.ts:35-36](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/secrets.ts#L35-L36)

## Criterion details

### C1 Identity & least privilege — 0.35 (high)

The server signs in as the user through Microsoft's login library and holds one delegated token for everything. It does request only the Graph permissions needed by the tools that are switched on, and operators can narrow that with --read-only, presets or an allowed-scopes list. Out of the box, though, that one token can read and write the user's mail, calendar, OneDrive, contacts, tasks and notes and can send mail. Nothing checks individual requests against a policy. If the model is hijacked, it has the user's whole personal Microsoft 365 account.

- **S L2:** Scopes are derived from the enabled tool set (least privilege per tool set), but one delegated token with read and write scopes serves every tool for the whole run. — [src/auth.ts:437-476](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/auth.ts#L437-L476); [src/endpoints.json:80-85](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L80-L85) (verified)
  - *To reach the next level:* No per-tool or read/write credential separation and no per-request authorization gate; read tools use the same ReadWrite/Mail.Send token.
- **C L2:** Every Graph and utility tool goes through AuthManager's single token (or the per-account token the model selects); there are no extensions or sub-agents that bypass it, and no per-request authorization layer exists to cover. — [src/auth-tools.ts:183-191](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/auth-tools.ts#L183-L191); [src/graph-tools.ts:1969](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L1969) (verified)
  - *To reach the next level:* No authorization layer in code that every tool call passes through; the model can pick among cached accounts via select-account or the account parameter.
- **D L1:** The default personal mode is narrower than --org-mode but still requests Mail.ReadWrite, Mail.Send, Files.ReadWrite, Calendars.ReadWrite and other write scopes; least privilege needs --read-only or a preset. — [src/auth.ts:437-476](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/auth.ts#L437-L476); [src/endpoints.json:80-85](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L80-L85); [src/cli.ts:34](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/cli.ts#L34) (verified)
  - *To reach the next level:* Default should be read-only or near-minimal, with write scopes needing an explicit operator flag.
- **B L0:** A hijacked session holds delegated write access across mail (including send), OneDrive, calendar, contacts, To Do and OneNote with a long-lived refresh token, which is effectively the user's entire personal M365 account. — [src/endpoints.json:80-85](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L80-L85); [src/endpoints.json:639](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L639); [src/auth.ts:437-476](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/auth.ts#L437-L476) (verified)
  - *To reach the next level:* Blast radius would need to be one service or mostly-read scopes.
- **Cap:** none
- **Notes:** HTTP mode (not scored) accepts a client bearer token and forwards it to Graph unless --obo is used; that would bring C1-PASSTHRU into play for HTTP deployments.

### C2 Approval gates — 0.25 (high)

The server has no approval step of its own; it relies on the host to ask the user, and gives the host risk labels for each tool. Labels for the 300+ Graph tools follow the HTTP method and are accurate, but one local-file tool's label is inaccurate, the account-management tools have no labels, and graph-batch can mix reads and writes in one call. The optional confirm gate is off by default and is satisfied by the model passing confirm: true, so it is not human approval. By default the model can send email and delete items without any server-side check.

- **S L1:** Graph tools carry readOnlyHint/destructiveHint derived from the HTTP method, but one local-file tool's annotation is inaccurate, logout/select-account/remove-account carry no annotations, and graph-batch mixes reads and writes. — [src/graph-tools.ts:2882-2898](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L2882-L2898); [src/auth-tools.ts:84](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/auth-tools.ts#L84); [src/endpoints.json:3-7](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L3-L7) (verified)
  - *To reach the next level:* Accurate annotations on every tool and separate read and write tools.
- **C L1:** Annotations are generated for every Graph tool in one place, but annotation and read-only-mode coverage are not complete. (verified)
  - *To reach the next level:* Every mutating path, including utility and auth tools, must be flagged so a host gate catches it.
- **D L2:** Annotations are always emitted and cannot be changed by the model; the server-side confirm gate is opt-in via MS365_MCP_REQUIRE_CONFIRM and is satisfiable by a model-supplied confirm:true. — [src/graph-tools.ts:487-489](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L487-L489); [src/graph-tools.ts:1914-1921](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L1914-L1921) (verified)
  - *To reach the next level:* A server-side gate that is on by default and requires a principal, not a model argument, to satisfy.
- **B L0:** send-mail and other send tools are enabled by default and are irreversible; deletes are only partly recoverable through Graph's own recycle bins. — [src/endpoints.json:83](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L83); [src/endpoints.json:639](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L639) (verified)
  - *To reach the next level:* Previews/dry-runs for external actions, or send tools off by default.
- **Cap:** none
- **Notes:** The opt-in confirm gate is model-satisfiable (params.confirm === true); it would trigger C2-SELFAPPROVE if it were the primary control. Message sign-off markers (--message-signoff-prefix/suffix) exist but default to none.

### C3 Tool & action scoping — 0.45 (high)

Most tools are narrow, one per Microsoft Graph endpoint, with typed schemas, URL-encoded path values and a fixed Graph host. Three general tools widen that: graph-batch sends up to 20 arbitrary Graph requests of any method, download-bytes reads any Graph path, and download-bytes-to-file writes downloaded bytes to any absolute path the model chooses. That last one only refuses to overwrite and uses owner-only permissions; it has no directory restriction. Presets, regex filters and --read-only can shrink the tool set, but by default every personal-mode write tool is on.

- **S L2:** Typed zod schemas, encodeURIComponent on path parameters and a fixed Graph base URL, but graph-batch forwards arbitrary sub-request methods/URLs and download-bytes-to-file accepts any absolute output path (only O_EXCL no-overwrite). — [src/graph-tools.ts:2083-2085](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L2083-L2085); [src/graph-client.ts:502](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-client.ts#L502); [src/endpoints.json:3-7](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L3-L7); [src/graph-tools.ts:1384](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L1384); [src/graph-client.ts:423](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-client.ts#L423) (verified)
  - *To reach the next level:* Allowlist validation: contain download-bytes-to-file to a configured directory and restrict graph-batch sub-requests to enabled tools' paths/methods.
- **C L2:** Schema validation applies to every generated Graph tool; graph-batch sub-requests and the local output path are not validated beyond shape. — [src/endpoints.json:3-7](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L3-L7); [src/graph-tools.ts:1384](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L1384) (verified)
  - *To reach the next level:* A central policy layer that validates every tool's arguments, including batch sub-requests.
- **D L2:** Presets, --enabled-tools regex and --read-only exist, but the default personal-mode set includes send, delete, file-upload and the local file writer. — [src/cli.ts:81-84](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/cli.ts#L81-L84); [src/cli.ts:34](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/cli.ts#L34); [src/graph-tools.ts:1310](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L1310) (verified)
  - *To reach the next level:* Read-only tool set by default with write tools enabled explicitly.
- **B L1:** graph-batch reaches any Graph endpoint the token covers, and download-bytes-to-file can create files anywhere the user can write; limits are host-fixed Graph and no-overwrite. — [src/endpoints.json:3-7](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L3-L7); [src/graph-client.ts:423](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-client.ts#L423); [src/graph-tools.ts:1384](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L1384) (verified)
  - *To reach the next level:* Tools scoped to a workspace or bounded quantities (recipient caps, output directory).
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

The server never runs model-generated code or commands. The only child process is an optional token-cache helper that the operator configures through an environment variable as an absolute path, run without a shell; it cannot be set from a project .env file or by the model. The local file writer is scored under tool scoping; the server itself interprets nothing as code.

- **Structural absence:** searched `rg -n "child_process|execFile|spawn\(|eval\(|new Function|vm\.runIn"` in `src` → 1 hits (The single hit is the import for the operator-configured MS365_MCP_AUTH_CACHE_COMMAND helper (src/token-cache-storage.ts:999, absolute path required at :1066, shell:false); it is set only from the real environment, not from .env (src/load-env.ts:23-28), and no tool reaches it.)
- **Notes:** Local file writes are scored under C3/C5, not here.

### C5 Untrusted input blast radius — 0.25 (high)

The server feeds the model content anyone can write: inbound email, calendar invites, shared files and OneNote pages. Results come back as structured Graph JSON, but nothing marks them as untrusted or limits what the model can do afterwards. In the same default session the model can read private mail and files, send email to any address, delete items, and write attachment bytes to the local disk. One malicious email can therefore lead to data leaving by mail and to irreversible actions with no server-side check.

- **S L2:** Tool results are structured Graph JSON with transport metadata kept separately in _meta, so content is not mixed with server instructions. — [src/graph-client.ts:308](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-client.ts#L308); [src/graph-tools.ts:2643-2646](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L2643-L2646) (verified)
  - *To reach the next level:* Provenance on returned content (an untrusted flag or source marker the host can act on).
- **C L2:** Every Graph tool returns through the same JSON path; download-bytes returns raw text/base64 without structure beyond the envelope. — [src/graph-client.ts:308](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-client.ts#L308); [src/graph-tools.ts:1281-1285](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L1281-L1285) (verified)
  - *To reach the next level:* Every source tagged as untrusted, including binary/raw downloads.
- **D L3:** Structured output is always on; the only alternative format (TOON) is also structured and opt-in, and nothing read from content changes it. — [src/graph-client.ts:308](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-client.ts#L308); [src/cli.ts:92](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/cli.ts#L92) (verified)
  - *To reach the next level:* Nothing to switch off, but there is no provenance control for content to be unable to disable either.
- **B L0:** Default stdio session combines untrusted inbound mail (list-mail-messages), private data (Mail.Read, Files.Read) and unattended send-mail/delete tools, with no server-side approval. — [src/endpoints.json:12](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L12); [src/endpoints.json:83](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L83); [src/endpoints.json:639](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L639); [src/graph-tools.ts:487-489](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L487-L489) (verified)
  - *To reach the next level:* Drop one Rule-of-Two leg by default (e.g., read-only unless the operator enables writes) or require a principal-approved confirmation for send/delete.
- **Cap:** C5-WORSTCASE — In the default configuration a hijacked session can both exfiltrate mail/files (send-mail to any address) and take irreversible actions without human involvement.

### C6 Memory, context & configuration integrity — 0.25 (high)

The server keeps no memory and loads no instruction files. It does read a .env file from whatever directory it is started in, which MCP clients often set to the open project. After a past advisory this was narrowed to four keys, but those keys are the app registration's client ID and secret, the tenant and the Microsoft cloud. A cloned repository can therefore silently point the sign-in at a different app registration, tenant or cloud whenever the operator has not set them.

- **S L1:** A cwd .env is parsed and allowlisted to four keys, which are applied silently, with no trust prompt, whenever the operator environment leaves them unset. — [src/load-env.ts:23-28](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/load-env.ts#L23-L28); [src/load-env.ts:43](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/load-env.ts#L43); [src/load-env.ts:64](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/load-env.ts#L64) (verified)
  - *To reach the next level:* Security-relevant identity settings (client ID, tenant, cloud) should come only from user/operator scope or behind an explicit trust decision.
- **C L2:** The one auto-loaded file is covered by the allowlist; no other workspace file is read at startup. — [src/index.ts:43](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/index.ts#L43); [src/load-env.ts:23-28](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/load-env.ts#L23-L28) (verified)
  - *To reach the next level:* All auto-loaded settings controlled, including the four identity keys.
- **D SA:** No memory store, vector index or cross-session conversation persistence exists. — searched `rg -n -i "vector|embedding|remember|save_memory|saveMemory"` in `src` → 9 hits (Hits are comments and endpoints.json llmTip text ('Remember: ALWAYS wrap...'); none is a memory store.) (verified)
- **B L2:** A planted .env persists for every launch in that directory and changes which app registration/tenant/cloud the user signs into, but cannot add tools or trigger tool calls. — [src/secrets.ts:35-36](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/secrets.ts#L35-L36); [src/load-env.ts:64](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/load-env.ts#L64) (verified)
  - *To reach the next level:* Persistence scoped so it is easily inspected and purged, or ignored without trust.
- **Cap:** C6-REPOCONFIG — A .env in the launch directory can, without a trust decision, set MS365_MCP_CLIENT_ID/TENANT_ID/CLIENT_SECRET/CLOUD_TYPE, redirecting the credential identity and Graph/login cloud.

### C7 Third-party extensions — 1.00 (high)

The server loads no plugins, remote tools or third-party MCP servers. Every tool comes from the bundled endpoints.json, and the only dynamic imports are its own packaged dependencies (keytar, Azure identity, the browser opener). Supply-chain risk in those dependencies is outside this criterion.

- **Structural absence:** searched `rg -n -i "plugin|loadExtension|mcpServers|marketplace"` in `src` → 14 hits (All hits are MSAL's token-cache plugin interface and comments about it; no extension loader.); searched `rg -n "import\("` in `src` → 11 hits (Dynamic imports are bundled dependencies (open, @azure/identity, @azure/keyvault-secrets, keytar) and test helpers; none loads user- or model-chosen code.)

### C8 Secrets & sensitive-data protection — 0.45 (high)

Tokens are handled well. The token cache is encrypted with AES-256-GCM, with the key kept in the OS keychain when one is available. Log output is scrubbed of tokens and email addresses by default, and tokens never appear in tool results. Gaps: redaction can be switched off with an environment variable, every tool's full arguments (message bodies included) are written to the operational log, crash dumps go to stderr unredacted, and get-download-url deliberately hands the model a pre-authenticated download link. The refresh token is long-lived and carries broad write scopes.

- **S L2:** Cache encrypted with AES-256-GCM, key in the OS keychain (falling back to a 0600 key file beside the cache), plus regex redaction of JWTs, bearer headers, OAuth fields and emails in every log message; nothing redacts model-bound tool results, and get-download-url returns a pre-authenticated URL to the model. — [src/token-cache-storage.ts:538](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/token-cache-storage.ts#L538); [src/token-cache-storage.ts:857](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/token-cache-storage.ts#L857); [src/lib/log-redactor.ts:27](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/lib/log-redactor.ts#L27); [src/logger.ts:12](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/logger.ts#L12) (verified)
  - *To reach the next level:* Redaction or capability-handle substitution before model-bound messages (pre-authenticated download URLs go to the model as-is).
- **C L2:** Redaction runs in the logger format for file and console transports and the audit log excludes params, but model-bound results are unfiltered and crash dumps go to stderr unredacted. — [src/logger.ts:12](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/logger.ts#L12); [src/index.ts:37](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/index.ts#L37) (verified)
  - *To reach the next level:* Cover model-bound messages and error handlers (the uncaughtException/unhandledRejection dumps print raw error properties to stderr).
- **D L2:** No telemetry; redaction on by default but disabled silently by MS365_MCP_REDACT_PII=false, and full tool params are logged at info level by default. — [src/lib/log-redactor.ts:61-64](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/lib/log-redactor.ts#L61-L64); [src/graph-tools.ts:1912](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L1912) (verified)
  - *To reach the next level:* Redaction always on and tool params minimised in default logs.
- **B L1:** The cached refresh token is long-lived and carries write scopes across mail, files and calendar; it is not reachable by the model. — [src/endpoints.json:80-85](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/endpoints.json#L80-L85); [src/token-cache-storage.ts:857](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/token-cache-storage.ts#L857) (verified)
  - *To reach the next level:* Short-lived or narrowly scoped credentials.
- **Cap:** none

### C9 Audit & traceability — 0.38 (high)

There is a separate JSON audit log, on by default and kept in the user's home directory with owner-only permissions. It records each Graph and utility tool call with a request ID, tool, HTTP method, status, duration, target resource and recipient domains, plus calls refused by tool filters. In the default local mode it records no user identity, because that is read from a bearer token only present in HTTP mode. Account tools such as select-account and logout, and calls refused by the confirm gate, are not recorded. Writes are best-effort, and a single environment variable turns the log off.

- **S L2:** Structured JSON record per tool call with timestamp, tool, method, status, duration, target resource, recipient domains and request_id; full arguments are only in the operational log. — [src/graph-tools.ts:2630-2641](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L2630-L2641); [src/graph-tools.ts:1942](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L1942) (verified)
  - *To reach the next level:* Actor attribution: user_principal_name is undefined in stdio mode because it is decoded only from request-context bearer tokens.
- **C L1:** Graph and utility tools and allowlist denials are audited, but the auth tools (login, logout, select-account, remove-account) and confirm-gate refusals bypass the audit path. — [src/graph-tools.ts:823-832](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L823-L832); [src/server.ts:324-326](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/server.ts#L324-L326); [src/graph-tools.ts:1914-1921](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L1914-L1921) (verified)
  - *To reach the next level:* Record every built-in tool, including account switching/removal and confirm refusals.
- **D L2:** On by default and written outside the workspace (~/.ms-365-mcp-server/logs, 0600); the model's only file tool cannot overwrite existing files, but MS365_MCP_AUDIT_LOG=false disables it silently. — [src/audit-log.ts:120-122](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/audit-log.ts#L120-L122); [src/audit-log.ts:44-45](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/audit-log.ts#L44-L45); [src/graph-client.ts:423](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-client.ts#L423) (verified)
  - *To reach the next level:* Disabling should require a logged operator action, written by a component the server process cannot alter.
- **B L1:** Winston file transport is asynchronous best-effort; directory-creation failures are swallowed and actions proceed regardless. — [src/audit-log.ts:50-60](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/audit-log.ts#L50-L60) (verified)
  - *To reach the next level:* Errors surfaced and records flushed per action.
- **Cap:** none

### C10 Limits & kill switch — 0.50 (high)

Every Graph call has a 100-second timeout, a capped number of retries and a circuit breaker. Paginated reads stop after 100 pages or 10,000 items, and all of these limits have sensible defaults the operator can change. Rate limiting exists only in HTTP mode, so the default local mode has none. Nothing stops a cancelled request from finishing, and file downloads to disk have no size cap.

- **S L2:** Server-enforced fetch timeout, retry cap, circuit breaker and page/item caps on fetchAllPages. — [src/lib/graph-resilience.ts:64-67](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/lib/graph-resilience.ts#L64-L67); [src/graph-tools.ts:2483-2491](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L2483-L2491); [src/graph-tools.ts:466](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-tools.ts#L466) (verified)
  - *To reach the next level:* Caps on every operation plus rate limits in stdio mode.
- **C L2:** Timeouts and the breaker wrap every Graph fetch via fetchWithResilience; rate limiting is HTTP-only. — [src/lib/graph-resilience.ts:64-67](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/lib/graph-resilience.ts#L64-L67); [src/server.ts:585-589](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/server.ts#L585-L589) (verified)
  - *To reach the next level:* Rate/concurrency limits covering all transports, including stdio.
- **D L2:** Defaults are sensible (100 s, 3 retries, 100 pages, 10,000 items) and operator-configurable through env vars; the model cannot raise them. — [src/lib/graph-resilience.ts:64-67](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/lib/graph-resilience.ts#L64-L67); [src/lib/param-descriptions.ts:45](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/lib/param-descriptions.ts#L45) (verified)
  - *To reach the next level:* Hard ceilings that configuration cannot exceed.
- **B L2:** Moderate per-call ceilings; no cancellation of in-flight work and no size cap on download-bytes-to-file streams. — [src/lib/graph-resilience.ts:64-67](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/lib/graph-resilience.ts#L64-L67); [src/graph-client.ts:423](https://github.com/Softeria/ms-365-mcp-server/blob/c47197109dcf8ea6cea91ee08f7417adfbeac72b/src/graph-client.ts#L423) (verified)
  - *To reach the next level:* Cancellation of pending calls on stop and tight per-run ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Inbound mail via list-mail-messages (src/endpoints.json:12), shared files, calendar invites · [B] sensitive data/systems: Mail.Read/Files.ReadWrite delegated token (src/auth.ts:437-476) · [C] state change / egress: send-mail (src/endpoints.json:83), deletes, download-bytes-to-file · Same default session? Yes

## Highest-impact improvements
1. Audit risk annotations and read-only-mode coverage for every tool; annotate the auth tools. — C2 S L1→L2, +0.075 before caps (Playbook 5)
2. Confine download-bytes-to-file to a configured download directory (realpath containment) and restrict graph-batch sub-requests to enabled tools' paths and methods. — C3 S L2→L3, +0.075 before caps (Playbook 3)
3. Stop reading MS365_MCP_CLIENT_ID/TENANT_ID/CLIENT_SECRET/CLOUD_TYPE from a cwd .env (or require an explicit opt-in flag). — C6 S L1→L3, +0.150 before caps (Playbook 2)
4. Audit the auth tools and confirm-gate refusals, and record the MSAL account username as actor in stdio mode. — C9 C L1→L2, +0.075 before caps
5. Ship read-only as the default, with writes enabled by an explicit operator flag. — C1 D L1→L3, +0.100 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- HTTP transport, OBO, --trust-proxy-auth, discovery mode and the attachment-URL listener were reviewed only at a high level; scores describe stdio defaults.
- Third-party library behaviour (MSAL, undici redirect handling, winston flushing, keytar) is assumed from their documented behaviour, not re-verified.
- Generated client code (src/generated) and endpoints.json were sampled, not exhaustively reviewed per endpoint.
- No reviewer-steering or prompt-injection text aimed at auditors was found in the repository.
