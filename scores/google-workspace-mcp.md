# Defense-in-Depth Score: Google Workspace MCP

**Repo:** https://github.com/taylorwilsdon/google_workspace_mcp · **Commit:** `3ada8badf7efe7f87c864ec58ceb07d01f0f49d9` (v2.0.0) · **Reviewed:** 2026-10-03
**What it is:** MCP server controlling Gmail, Calendar, Docs, Sheets, Drive, Chat etc.
**Category:** AI Assistants
**Scored configuration:** `uvx workspace-mcp` with no flags: stdio transport, all 12 services and every tool loaded, legacy OAuth 2.0 with full read/write scopes, local credential files.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 5.4 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L0 | L2 | 0.40 | G1 | **0.40** (alt) | High |
| C2 | Approval gates | L2 | L2 | L3 | L0 | 0.45 | — | **0.45** | High |
| C3 | Tool & action scoping | L3 | L2 | L2 | L1 | 0.53 | — | **0.53** | High |
| C4 | Code-execution isolation | L4 | L4 | L3 | L1 | 0.80 | — | **0.80** | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C9 | Audit & traceability | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |

Controls where a risk surface exists: 3.40 / 8.0 (42%); 2 criteria scored SA (surface absent).

Run with no flags, this server gives the model write access to the user's whole Google Workspace account, including sending email, sharing files publicly and writing and running Apps Script. Local file and URL handling is well hardened, and it runs no code on the host. The dominant risk is prompt injection: emails and documents it reads come back as plain text beside its own instructions, and nothing in the server stops an injected instruction from leaking data or sending mail. Use --read-only or --permissions to cut its authority.

## Critical gaps
- Default install requests write scopes across the user's entire Google Workspace account (mail send, full Drive, Apps Script execution), so any control failure exposes the whole account. (ASI03, T3, LLM06; C1) — [auth/scopes.py:338-340](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/scopes.py#L338-L340); [auth/scopes.py:195-205](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/scopes.py#L195-L205)
- Untrusted email/document content, the user's private Workspace data, and send/share tools coexist in one default session with nothing in the server separating them. (ASI01, T6, LLM01; C5) — [gmail/gmail_tools.py:2005](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gmail/gmail_tools.py#L2005); [gmail/gmail_tools.py:2527-2537](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gmail/gmail_tools.py#L2527-L2537); [gdrive/drive_tools.py:2725-2736](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gdrive/drive_tools.py#L2725-L2736)

## Criterion details

### C1 Identity & least privilege — 0.40 (high)

Out of the box the server asks Google for one OAuth grant that covers every enabled service with full write scopes: all of Drive, Gmail send/modify/settings, Calendar, Contacts, Chat, and Apps Script project, deployment and external-request scopes. Each tool only checks that the stored token has enough scopes; it then receives the same all-powerful token. In the default local mode the target account is a tool argument the model fills in, so whichever stored account it names is used. An opt-in --permissions flag requests per-service minimal scopes and removes tools that need more, which is the scored mechanism here, but it is off by default.

- **default configuration** (default; raw 0.23 → 0.23)
  - **S L1:** A single user OAuth grant with the union of every service's write scopes is used for every tool; per-tool scopes are only checked, never used to narrow the token. — [auth/scopes.py:338-340](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/scopes.py#L338-L340); [auth/scopes.py:156-163](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/scopes.py#L156-L163); [auth/google_auth.py:1287](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/google_auth.py#L1287) (verified)
    - *To reach the next level:* Request read and write scopes separately (or per capability) instead of one grant for the whole Workspace account.
  - **C L2:** Every Google tool goes through require_google_service and the same credential lookup, but in default legacy mode the account is chosen by the model-supplied user_google_email argument rather than a verified principal. — [auth/service_decorator.py:860](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/service_decorator.py#L860); [auth/service_decorator.py:543](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/service_decorator.py#L543); [auth/google_auth.py:1168](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/google_auth.py#L1168) (verified)
    - *To reach the next level:* Bind each call to an authenticated requesting principal instead of a model-supplied email argument.
  - **D L0:** With no flags every service is imported and the full write scope set is requested. — [main.py:759-763](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/main.py#L759-L763); [auth/scopes.py:338-340](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/scopes.py#L338-L340) (verified)
    - *To reach the next level:* Ship with read-only or minimal scopes by default and require explicit operator elevation for write and send.
  - **B L0:** A hijacked server holds read/write over the user's entire Google Workspace account: mail send, Drive sharing, calendar, contacts and Apps Script. — [auth/scopes.py:156-163](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/scopes.py#L156-L163); [auth/scopes.py:195-205](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/scopes.py#L195-L205) (verified)
    - *To reach the next level:* Limit default authority to one service or read-only scopes so a hijack cannot reach the whole account.
- **opt-in --permissions service:level granular scopes** (alt; raw 0.40, cap G1 → 0.40) ← counted
  - **S L2:** --permissions maps each service level to a fixed cumulative scope list, so the OAuth grant is limited to the chosen services and levels. — [auth/permissions.py:66-73](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/permissions.py#L66-L73); [auth/scopes.py:327-334](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/scopes.py#L327-L334); [main.py:508-519](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/main.py#L508-L519) (verified)
    - *To reach the next level:* Read and write within a service still share one token; separate per-capability credentials are not issued.
  - **C L2:** Tools whose required scopes exceed the configured levels are removed from the server at startup. — [core/tool_registry.py:202-221](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/core/tool_registry.py#L202-L221); [auth/service_decorator.py:543](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/service_decorator.py#L543) (verified)
    - *To reach the next level:* No per-request principal check in legacy mode; the model-supplied email still selects the account.
  - **D L0:** Granular permissions are only active when the operator passes --permissions or WORKSPACE_MCP_PERMISSIONS. — [main.py:508-519](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/main.py#L508-L519); [auth/scopes.py:338-340](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/scopes.py#L338-L340) (verified)
    - *To reach the next level:* Make a minimal permission set the default.
  - **B L2:** With e.g. gmail:readonly drive:readonly, a hijack is limited to reading the selected services. — [auth/permissions.py:66-73](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/permissions.py#L66-L73) (verified)
    - *To reach the next level:* Short-lived, task-scoped tokens would bound blast radius further.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.
- **Notes:** OAuth 2.1 / external-provider HTTP mode (opt-in) accepts Google access tokens from the client and uses them downstream; that mode was not scored.

### C2 Approval gates — 0.45 (high)

As a tool server it relies on the MCP host to ask the user before acting; what it provides is a read-only or destructive label on every one of its tools, and those labels are mostly accurate. There is no preview or dry-run for sending mail, changing sharing, or deleting, except a dry-run for applying Gmail filters. A server-enforced read-only mode exists but must be switched on. If a host auto-approves or the user approves the wrong call, emails, public file shares and calendar invites go out immediately and cannot be recalled.

- **S L2:** Reads and writes are separate tools and every tool carries readOnlyHint/destructiveHint annotations. — [gmail/gmail_tools.py:2527-2537](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gmail/gmail_tools.py#L2527-L2537); [gdrive/drive_tools.py:2725-2736](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gdrive/drive_tools.py#L2725-L2736); searched `rg -n -i 'dry_run|dry-run'` in `auth core gmail gdrive gdocs gsheets gslides gcalendar gchat gcontacts gforms gtasks gsearch gappsscript main.py fastmcp_server.py` → 11 hits (all hits are manage_gmail_filter's apply action; no preview for send, share, delete or script tools) (verified)
  - *To reach the next level:* No preview or dry-run for destructive operations other than Gmail filter apply.
- **C L2:** All ~130 tools are annotated, but get_drive_file_download_url is marked readOnlyHint=True while saving files to local disk, and send_gmail_message is marked destructiveHint=False. — [gdrive/drive_tools.py:444-451](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gdrive/drive_tools.py#L444-L451); [gdrive/drive_tools.py:464](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gdrive/drive_tools.py#L464); [gmail/gmail_tools.py:2527-2537](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gmail/gmail_tools.py#L2527-L2537) (verified)
  - *To reach the next level:* Make every hint reflect side effects so a host gating on them misses nothing.
- **D L3:** Annotations are hard-coded on each tool and cannot be changed by the model or by content. — [gmail/gmail_tools.py:2527-2537](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gmail/gmail_tools.py#L2527-L2537); [main.py:503-507](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/main.py#L503-L507) (verified)
  - *To reach the next level:* Server-side read-only mode is opt-in; no time-bounded elevation or principal-bound confirmation.
- **B L0:** A wrongly approved call can send or forward email, share files with anyone, or delete Drive, calendar and contact data with no undo. — [gmail/gmail_tools.py:2527-2537](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gmail/gmail_tools.py#L2527-L2537); [gdrive/drive_tools.py:2725-2736](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gdrive/drive_tools.py#L2725-L2736); searched `rg -n -i 'max_recipients|recipient_limit'` in `auth core gmail gdrive gdocs gsheets gslides gcalendar gchat gcontacts gforms gtasks gsearch gappsscript main.py fastmcp_server.py` → 0 hits (verified)
  - *To reach the next level:* Bound quantities (recipients, shares) and provide undo or previews for external actions.
- **Cap:** none

### C3 Tool & action scoping — 0.53 (high)

Arguments that touch the local machine or arbitrary URLs are validated well: local file paths are resolved and confined to the attachment directory by default, and URL fetches block private addresses and recheck every redirect. Most other arguments are typed and passed to Google's own APIs. However, with no flags every tool is loaded, including email send, file sharing, and Apps Script tools that write and run code, and nothing bounds recipients or the number of operations.

- **S L3:** validate_file_path resolves symlinks and checks containment under an allowlist; ssrf_safe_stream validates each resolved IP and every redirect hop manually. — [core/utils.py:284](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/core/utils.py#L284); [core/utils.py:378-382](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/core/utils.py#L378-L382); [core/http_utils.py:75](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/core/http_utils.py#L75); [core/http_utils.py:280](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/core/http_utils.py#L280) (verified)
  - *To reach the next level:* Path checks are not a strict boundary and general tools such as Apps Script code authoring/execution remain.
- **C L2:** Every local-path and remote-URL argument goes through these validators; other arguments rely on typed schemas and Google-side validation. — [gdrive/drive_helpers.py:888](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gdrive/drive_helpers.py#L888); [core/utils.py:378-382](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/core/utils.py#L378-L382); searched `rg -n -i 'max_recipients|recipient_limit'` in `auth core gmail gdrive gdocs gsheets gslides gcalendar gchat gcontacts gforms gtasks gsearch gappsscript main.py fastmcp_server.py` → 0 hits (verified)
  - *To reach the next level:* No shared bounds on recipients, counts or batch sizes across tools.
- **D L2:** Tool groups are selectable (--tools, --tool-tier, --read-only, --disabled-tools) but the no-flag default loads every tool including send, share and script execution. — [main.py:759-763](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/main.py#L759-L763); [main.py:503-507](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/main.py#L503-L507) (verified)
  - *To reach the next level:* Default to a read-only tool set and require explicit enabling of write tools.
- **B L1:** A misused tool reaches the user's whole Workspace (any recipient, public sharing, Apps Script with any-host egress), limited mainly by Apps Script execution needing a manual shared-Cloud-project setup. — [auth/scopes.py:195-205](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/scopes.py#L195-L205); [gappsscript/apps_script_tools.py:518-520](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gappsscript/apps_script_tools.py#L518-L520); [gdrive/drive_tools.py:2725-2736](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gdrive/drive_tools.py#L2725-L2736) (verified)
  - *To reach the next level:* Scope tools to specific folders/calendars and bound quantities.
- **Cap:** none

### C4 Code-execution isolation — 0.80 (medium)

The server never runs code on the host: there is no shell, eval or subprocess path. The only code-execution path is Apps Script, where the model can write script code and run it through Google's Execution API, so it runs in Google's managed runtime rather than on the machine. Inside that runtime the code acts with the user's authorized scopes and can make outbound web requests, and the model can also edit the script manifest that declares those scopes.

- **S L4:** Model-authored Apps Script code is executed only via Google's remote scripts.run API, never locally. — [gappsscript/apps_script_tools.py:524-526](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gappsscript/apps_script_tools.py#L524-L526); [gappsscript/apps_script_tools.py:356-367](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gappsscript/apps_script_tools.py#L356-L367) (verified)
- **C L4:** No local execution path exists to bypass the remote runtime. — [gappsscript/apps_script_tools.py:524-526](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gappsscript/apps_script_tools.py#L524-L526); searched `rg -n 'subprocess|os\.system|os\.popen|\beval\(|\bexec\(|pickle|shell=True'` in `auth core gmail gdrive gdocs gsheets gslides gcalendar gchat gcontacts gforms gtasks gsearch gappsscript main.py fastmcp_server.py` → 8 hits (all 8 hits are in gappsscript/TESTING.md prose about a manual test token pickle; no code path) (verified)
- **D L3:** Remote execution is the only mode and cannot be switched to host execution, but the model can rewrite the script manifest via manage_script_content. — [gappsscript/apps_script_tools.py:356-367](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gappsscript/apps_script_tools.py#L356-L367); [gappsscript/apps_script_tools.py:524-526](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gappsscript/apps_script_tools.py#L524-L526) (verified)
  - *To reach the next level:* Keep the script's manifest/policy outside what the model can write.
- **B L1:** Script code runs with the user's Google scopes and the server requests script.external_request, so code has user-level Workspace authority plus outbound HTTP. — [auth/scopes.py:195-205](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/scopes.py#L195-L205); [gappsscript/apps_script_tools.py:518-520](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gappsscript/apps_script_tools.py#L518-L520) (inferred)
  - *To reach the next level:* Run scripts with narrowed scopes and no external-request egress.
- **Cap:** none
- **Notes:** Apps Script runtime properties (egress via UrlFetchApp, scope enforcement) are Google behaviour inferred from the scopes the server requests.

### C5 Untrusted input blast radius — 0.00 (high)

The server hands back email bodies, documents, chat messages and search results as plain text mixed with its own instructions to the model (for example 'Use get_gmail_attachment_content(...) to download'), with no marker that the content came from a third party. In the default configuration the same session can read an attacker's email, read private Drive and Gmail data, and send email or share files publicly. Nothing in the server breaks that combination, so a successful prompt injection can leak data and take irreversible actions unless the host stops it.

- **S L0:** Tool outputs are plain strings that interleave third-party content with directives addressed to the model. — [gmail/gmail_tools.py:1993](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gmail/gmail_tools.py#L1993); [gmail/gmail_tools.py:2005](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gmail/gmail_tools.py#L2005); searched `rg -n -i 'untrusted|prompt.injection|provenance'` in `auth core gmail gdrive gdocs gsheets gslides gcalendar gchat gcontacts gforms gtasks gsearch gappsscript main.py fastmcp_server.py` → 1 hits (single hit is a CORS origin check docstring in core/server.py; nothing marks returned content as untrusted) (verified)
  - *To reach the next level:* Return structured outputs that separate content from metadata, with provenance and an untrusted flag.
- **C L0:** No untrusted source (mail, docs, chat, search, form responses) is distinguished. — searched `rg -n -i 'untrusted|prompt.injection|provenance'` in `auth core gmail gdrive gdocs gsheets gslides gcalendar gchat gcontacts gforms gtasks gsearch gappsscript main.py fastmcp_server.py` → 1 hits (single hit is a CORS origin check docstring in core/server.py; nothing marks returned content as untrusted) (verified)
  - *To reach the next level:* Mark every third-party content source as untrusted in outputs.
- **D L0:** No untrusted-content control exists to be on by default. — [main.py:759-763](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/main.py#L759-L763); searched `rg -n -i 'untrusted|prompt.injection|provenance'` in `auth core gmail gdrive gdocs gsheets gslides gcalendar gchat gcontacts gforms gtasks gsearch gappsscript main.py fastmcp_server.py` → 1 hits (single hit is a CORS origin check docstring in core/server.py; nothing marks returned content as untrusted) (verified)
  - *To reach the next level:* Ship a default that drops a Rule-of-Two leg (e.g. read-only) when untrusted content is read.
- **B L0:** A hijacked session can read private mail/Drive and send email or share files publicly with no server-side gate. — [gmail/gmail_tools.py:2527-2537](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gmail/gmail_tools.py#L2527-L2537); [gdrive/drive_tools.py:2725-2736](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gdrive/drive_tools.py#L2725-L2736); [auth/scopes.py:338-340](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/scopes.py#L338-L340) (verified)
  - *To reach the next level:* Make egress and state change unavailable or gated in sessions that read untrusted content.
- **Cap:** C5-WORSTCASE — In the default configuration an injected email can drive data exfiltration (send/forward/share) and irreversible actions with no server-side check.
- **Notes:** The opt-in --read-only mode removes all write/send tools and would drop the state-change leg; it is not on by default.

### C6 Memory, context & configuration integrity — 1.00 (high)

The server keeps no memory, retrieval store or conversation history that is fed back to the model, and it does not load instruction or configuration files from the user's working directory; its .env is read only from its own install directory. Stored OAuth credentials and downloaded attachments are not read back into model context as instructions.

- **Structural absence:** searched `rg -n -i 'vector.?store|save_memory|chromadb|faiss|AGENTS\.md|CLAUDE\.md'` in `auth core gmail gdrive gdocs gsheets gslides gcalendar gchat gcontacts gforms gtasks gsearch gappsscript main.py fastmcp_server.py` → 0 hits; searched `rg -n 'load_dotenv\('` in `auth core gmail gdrive gdocs gsheets gslides gcalendar gchat gcontacts gforms gtasks gsearch gappsscript main.py fastmcp_server.py` → 2 hits (main.py and fastmcp_server.py, both load .env from the package's own directory, not the working directory)

### C7 Third-party extensions — 1.00 (high)

The server loads no plugins, extensions, remote tools or model files at runtime; service modules are imported from a fixed in-repo list. Its own Python dependencies are build-time supply chain and out of scope here.

- **Structural absence:** searched `rg -n 'entry_points|import_module|pip install|npx'` in `auth core gmail gdrive gdocs gsheets gslides gcalendar gchat gcontacts gforms gtasks gsearch gappsscript main.py fastmcp_server.py` → 2 hits (both hits are main.py importing the fixed first-party SERVICE_MODULES map); searched `rg -n 'subprocess|os\.system|os\.popen|\beval\(|\bexec\(|pickle|shell=True'` in `auth core gmail gdrive gdocs gsheets gslides gcalendar gchat gcontacts gforms gtasks gsearch gappsscript main.py fastmcp_server.py` → 8 hits (all 8 hits are in gappsscript/TESTING.md prose about a manual test token pickle; no code path)

### C8 Secrets & sensitive-data protection — 0.45 (high)

OAuth tokens never go into tool output sent to the model, HTTP client logs that would print access tokens are silenced, and URL query strings are scrubbed from error logs. Refresh tokens, however, are stored as plaintext JSON files (owner-only permissions) and are long-lived and broadly scoped. Telemetry is off unless an OpenTelemetry endpoint is configured, and debug logging that includes user text is opt-in.

- **S L2:** Credential files are plaintext JSON written 0600; httpx logs are raised to WARNING and URL queries scrubbed from HTTP error logs. — [auth/credential_store.py:234](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/credential_store.py#L234); [auth/credential_store.py:223-226](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/credential_store.py#L223-L226); [main.py:98-101](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/main.py#L98-L101); [core/utils.py:1085-1093](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/core/utils.py#L1085-L1093) (verified)
  - *To reach the next level:* No OS keychain or encryption at rest for the local credential store.
- **C L2:** Logs and error messages are protected; there are no subprocesses; tokens are not returned in tool output. — [main.py:98-101](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/main.py#L98-L101); [core/utils.py:1085-1093](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/core/utils.py#L1085-L1093); searched `rg -n 'subprocess|os\.system|os\.popen|\beval\(|\bexec\(|pickle|shell=True'` in `auth core gmail gdrive gdocs gsheets gslides gcalendar gchat gcontacts gforms gtasks gsearch gappsscript main.py fastmcp_server.py` → 8 hits (all 8 hits are in gappsscript/TESTING.md prose about a manual test token pickle; no code path) (verified)
  - *To reach the next level:* No redaction pass on content sent back to the model (e.g. secrets inside emails or docs).
- **D L2:** OTel export is opt-in and the default log level is INFO; DEBUG (with user text) needs an env var. — [core/telemetry.py:113-114](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/core/telemetry.py#L113-L114); [main.py:105-111](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/main.py#L105-L111); [core/log_formatter.py:226-236](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/core/log_formatter.py#L226-L236) (verified)
  - *To reach the next level:* Redaction is not always-on: WORKSPACE_MCP_LOG_LEVEL=DEBUG writes user content to the log file.
- **B L1:** A leaked credential file is a long-lived refresh token for the user's full Workspace scopes, though not reachable by the model by default. — [auth/credential_store.py:223-226](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/credential_store.py#L223-L226); [auth/scopes.py:338-340](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/scopes.py#L338-L340) (verified)
  - *To reach the next level:* Use narrower or short-lived tokens so a leak expires or is confined.
- **Cap:** none

### C9 Audit & traceability — 0.38 (high)

Each tool call writes an unstructured text log line naming the tool, the Google account and the service, into a log file under the user's home directory that is on by default. Arguments and results are not recorded in a structured way, there is no approver or session attribution, and logging failures are only reported to stderr while the server keeps running. Structured OpenTelemetry tracing exists but is opt-in.

- **S L1:** Per-call INFO line '[tool] email -> service/version' plus ad-hoc tool log lines, as free text. — [auth/service_decorator.py:834-838](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/service_decorator.py#L834-L838); [core/log_formatter.py:226-236](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/core/log_formatter.py#L226-L236) (verified)
  - *To reach the next level:* No structured record of each call's arguments and result status.
- **C L2:** The audit line is emitted by the shared auth decorator, so every Google tool is covered. — [auth/service_decorator.py:834-838](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/service_decorator.py#L834-L838) (verified)
  - *To reach the next level:* Approvals, configuration changes and credential use are not recorded as audit events.
- **D L2:** File logging is on by default in ~/.google_workspace_mcp/logs, outside any workspace, but written by the same process. — [core/log_formatter.py:193](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/core/log_formatter.py#L193); [core/log_formatter.py:226-236](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/core/log_formatter.py#L226-L236) (verified)
  - *To reach the next level:* Write the record through a component the server process cannot alter.
- **B L1:** If log setup fails the server prints to stderr and continues; records are best-effort. — [core/log_formatter.py:248-253](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/core/log_formatter.py#L248-L253) (verified)
  - *To reach the next level:* Surface logging failures and flush a durable record per action.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

Some of the server's own work is bounded: Google API calls time out after 30 seconds, Office document expansion is capped at 25 MiB, and email bodies are truncated. File downloads are uncapped unless the operator sets a limit, and there are no rate limits on sending mail, sharing files or other side effects, so a runaway host loop can keep acting until Google's own quotas stop it.

- **S L2:** Server-enforced caps exist on some operations: 30s HTTP timeout, 25 MiB Office XML expansion, 20k-char body truncation. — [auth/google_auth.py:98](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/google_auth.py#L98); [core/file_limits.py:45](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/core/file_limits.py#L45); [gmail/gmail_tools.py:101](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gmail/gmail_tools.py#L101); searched `rg -n -i 'token.?bucket|RateLimiter|rate_limiter|calls_per'` in `auth core gmail gdrive gdocs gsheets gslides gcalendar gchat gcontacts gforms gtasks gsearch gappsscript main.py fastmcp_server.py` → 0 hits (verified)
  - *To reach the next level:* No caps on every operation and no rate limits on side-effecting tools.
- **C L2:** The HTTP timeout applies to every Google API call via the shared http builder. — [auth/google_auth.py:117](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/auth/google_auth.py#L117) (verified)
  - *To reach the next level:* No limits that span a session or cover concurrent work by default.
- **D L1:** Download size is uncapped by default and there is no default rate limit. — [core/file_limits.py:68-77](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/core/file_limits.py#L68-L77); searched `rg -n -i 'token.?bucket|RateLimiter|rate_limiter|calls_per'` in `auth core gmail gdrive gdocs gsheets gslides gcalendar gchat gcontacts gforms gtasks gsearch gappsscript main.py fastmcp_server.py` → 0 hits (verified)
  - *To reach the next level:* Set a default file-size cap and default rate limits.
- **B L1:** Nothing bounds how many emails, shares or edits a runaway loop makes before Google quotas intervene. — [gmail/gmail_tools.py:2527-2537](https://github.com/taylorwilsdon/google_workspace_mcp/blob/3ada8badf7efe7f87c864ec58ceb07d01f0f49d9/gmail/gmail_tools.py#L2527-L2537); searched `rg -n -i 'token.?bucket|RateLimiter|rate_limiter|calls_per'` in `auth core gmail gdrive gdocs gsheets gslides gcalendar gchat gcontacts gforms gtasks gsearch gappsscript main.py fastmcp_server.py` → 0 hits (verified)
  - *To reach the next level:* Add per-session ceilings on side-effecting calls.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Gmail bodies returned as plain text (gmail/gmail_tools.py:1993) · [B] sensitive data/systems: full Drive/Gmail/Contacts read scopes by default (auth/scopes.py:338-340) · [C] state change / egress: send_gmail_message and set_drive_file_permissions (gmail/gmail_tools.py:2527, gdrive/drive_tools.py:2725) · Same default session? Yes

## Highest-impact improvements
1. Default to read-only scopes (or a minimal --permissions set) and require explicit opt-in for write/send. — C1 D L0→L3, +0.150 before caps (Playbook 4)
2. Return structured tool output separating third-party content from metadata, with source and an untrusted flag, and drop model directives from content. — C5 S L0→L3, +0.225 before caps (Playbook 1)
3. Fix inaccurate annotations (local-disk writes under readOnlyHint, email send as non-destructive). — C2 C L2→L3, +0.075 before caps (Playbook 5)
4. Write a structured JSON audit record per call with arguments, result status and session id. — C9 S L1→L2, +0.075 before caps (Playbook 1 step 3)
5. Ship a default download size cap and rate limits on send/share tools. — C10 D L1→L2, +0.050 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the no-flag stdio default; the README's first example adds --tool-tier core, which still includes send_gmail_message and Drive writes. OAuth 2.1 / streamable-HTTP, external-provider (bearer-token passthrough), service-account (domain-wide delegation), stateless and Helm deployments were not scored.
- Apps Script runtime behaviour (scope enforcement, UrlFetchApp egress, shared Cloud project requirement) is Google's and was inferred, not verified.
- FastMCP framework internals (argument validation, OpenTelemetry span contents) were not reviewed.
- No reviewer-injection attempts found; .github/instructions/general.instructions.md is a benign code-review style guide.
