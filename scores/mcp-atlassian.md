# Defense-in-depth score: MCP Atlassian

**Repo:** https://github.com/sooperset/mcp-atlassian · **Commit:** `0a5d2427144d72f5aade07d4fabd385421cec401` · **Reviewed:** 2026-10-05
**What it is:** Python MCP server (FastMCP) exposing about 98 Jira and Confluence tools for Atlassian Cloud and Server/Data Center.
**Category:** AI Assistants
**Scored configuration:** Local stdio server launched as the README Quick Start shows (uvx mcp-atlassian) with Jira and Confluence URL, username and API token in the environment, and no other settings: TOOLSETS unset (all toolsets, including write tools), READ_ONLY_MODE off, no project or space filters.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 5.7 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L1 | L1 | 0.40 | G1 | **0.40** (alt) | High |
| C2 | Approval gates | L2 | L3 | L3 | L2 | 0.62 | none | **0.62** | High |
| C3 | Tool & action scoping | L2 | L2 | L1 | L1 | 0.40 | none | **0.40** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L2 | L1 | L2 | L0 | 0.33 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L1 | 0.45 | none | **0.45** | High |
| C9 | Audit & traceability | L0 | L1 | L1 | L0 | 0.12 | none | **0.12** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | none | **0.40** | High |

Controls where a risk surface exists: 2.65 / 7.0 (38%); 3 criteria scored SA (surface absent).

MCP Atlassian gives an AI assistant about 98 Jira and Confluence tools, and in the README setup it runs them all, writes included, with the user's own long-lived API token. It runs no code, keeps no memory and loads no plugins, and it labels every tool clearly as read-only or as a write, with a server-enforced read-only mode, but that mode is off by default. The dominant risk is prompt injection through issues, comments, pages or service-desk requests: a hijacked session can copy private data into pages or comments and delete issues with nothing in the server asking a human. There is also no audit record of the changes it makes.

## Critical gaps
- In the default configuration a session that reads other people's issues, comments or pages can also write, comment, upload and delete across the user's whole Jira and Confluence access, with no server-enforced human step. (ASI01, T6, LLM01; C5). Evidence: [src/mcp_atlassian/utils/toolsets.py:195-203](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/toolsets.py#L195-L203); [src/mcp_atlassian/utils/io.py:20](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/io.py#L20)

## Criterion details

### C1 Identity & least privilege: 0.40 (high confidence)

In the scored setup the server signs every Jira and Confluence call with the one API token or personal access token the user puts in its environment. Atlassian API tokens carry the user's full access to both products, and the server does no per-tool or per-request authorization of its own, so whatever the user can read or change, the connected model can too. OAuth 2.0 is an opt-in alternative that asks for named scopes (by default read and write on Jira work plus Confluence space summaries), but read and write still share one grant. The opt-in HTTP transport supports per-user tokens supplied in request headers; that mode was not the scored default.

- **default configuration** (default; raw 0.12 → 0.12)
  - **S L0:** Basic auth with the user's own API token (or a PAT on Server/Data Center) is the only identity in the default setup, used for every tool. Evidence: [README.md:36-46](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/README.md#L36-L46); [src/mcp_atlassian/jira/config.py:282-283](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/jira/config.py#L282-L283); [src/mcp_atlassian/jira/client.py:152-156](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/jira/client.py#L152-L156) (verified)
    - *To reach the next level:* No dedicated or narrowed identity: the server never requests a scoped token or separates read and write credentials.
  - **C L1:** All 98 tools obtain their client through the shared get_jira_fetcher/get_confluence_fetcher dependency, so every tool runs on the same configured credential; there is no subprocess or extension path. Evidence: searched `rg -n -S 'get_jira_fetcher\(ctx\)'` in `src/mcp_atlassian/servers/jira.py` → 63 hits (one call per Jira tool; all resolve the same global config in stdio mode); [src/mcp_atlassian/servers/dependencies.py:1053](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/servers/dependencies.py#L1053) (verified)
    - *To reach the next level:* No authorization layer in code that checks each request against a least-privilege policy before the credential is attached.
  - **D L0:** The documented install hands the server the user's full-account API token for both products; narrowing it depends on the user creating a scoped credential themselves. Evidence: [README.md:36-46](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/README.md#L36-L46) (verified)
    - *To reach the next level:* No narrower default (for example a read-only credential or scoped token requested by the server).
  - **B L1:** A hijacked session can read and write everything the user can reach in both Jira and Confluence (issues, pages, comments, attachments, page restrictions). Evidence: [src/mcp_atlassian/servers/main.py:922-923](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/servers/main.py#L922-L923); [src/mcp_atlassian/confluence/client.py:139-140](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/confluence/client.py#L139-L140) (verified)
    - *To reach the next level:* Authority spans write access in two systems rather than one project or space.
- **opt-in OAuth 2.0 with requested scopes** (alt; raw 0.40, cap G1 → 0.40) ← counted
  - **S L2:** The OAuth flow requests named scopes (default read:jira-work write:jira-work read:confluence-space.summary offline_access) and stores tokens in the OS keyring. Evidence: [src/mcp_atlassian/utils/oauth_setup.py:426-429](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/oauth_setup.py#L426-L429) (verified)
    - *To reach the next level:* Read and write share one grant for the whole run; no per-tool or downscoped tokens.
  - **C L2:** When OAuth is configured, every tool's client is built on the OAuth session. Evidence: [src/mcp_atlassian/jira/client.py:96-102](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/jira/client.py#L96-L102) (verified)
    - *To reach the next level:* No per-request authorization check in code.
  - **D L1:** OAuth must be set up explicitly and the scope list is whatever the operator passes. Evidence: [src/mcp_atlassian/utils/oauth_setup.py:426-429](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/oauth_setup.py#L426-L429) (verified)
    - *To reach the next level:* Not the default; the scope can be widened by an environment variable without warning.
  - **B L1:** The default grant still writes to Jira work items across all projects the user can reach. Evidence: [src/mcp_atlassian/utils/oauth_setup.py:426-429](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/oauth_setup.py#L426-L429) (verified)
    - *To reach the next level:* Writes are not limited to one project or space.
- **Cap:** G1: Opt-in mechanism: off in the scored default configuration.
- **Notes:** The opt-in HTTP transport accepts per-user Atlassian tokens in request headers and refuses unauthenticated requests unless ALLOW_GLOBAL_CRED_FALLBACK is set; it was not scored.

### C2 Approval gates: 0.62 (high confidence)

As a tool server, this project leaves approval to the MCP host, and what it gives the host is good risk signalling: all 98 tools are split into separate read and write tools, every read tool is marked read-only and every write tool carries a destructive or non-destructive hint. A server-enforced read-only mode removes write tools both from the listing and at call time, and every write tool also checks it. There is no preview or dry-run for destructive operations except a validate-only option on batch issue creation, and the read-only mode is off by default. Deleted Jira issues and sent comments or notifications cannot be undone.

- **S L2:** Read and write tools are separate, with readOnlyHint on all 58 read tools and destructiveHint on the write tools (delete, move, update and transition marked destructive). Evidence: searched `rg -n -F '"destructiveHint": True'` in `src/mcp_atlassian/servers/` → 24 hits (one per destructive write tool (deletes, transitions, overwrites)); searched `rg -n -F '"readOnlyHint": True'` in `src/mcp_atlassian/servers/` → 58 hits (one per read tool (63 Jira + 35 Confluence tools = 58 read + 40 write)); [src/mcp_atlassian/servers/jira.py:1938](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/servers/jira.py#L1938) (verified)
  - *To reach the next level:* No preview or dry-run for destructive operations such as deleting issues, pages or attachments; only batch issue creation offers validate_only.
- **C L3:** Every write tool is tagged write and wrapped in check_write_access, and the tool filter applies the same authorization at listing and call time, rejecting unknown tools. Evidence: searched `rg -n -S '@check_write_access'` in `src/mcp_atlassian/servers/` → 40 hits (matches the 40 tools tagged write); [src/mcp_atlassian/servers/main.py:395-403](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/servers/main.py#L395-L403); [src/mcp_atlassian/servers/main.py:308-309](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/servers/main.py#L308-L309) (verified)
  - *To reach the next level:* Write tools are not individually confirmable by the server (no server-side confirmation step for destructive calls).
- **D L3:** Risk annotations are static in code and nothing at runtime alters them; the read-only mode is an operator setting outside model reach. Evidence: [src/mcp_atlassian/utils/io.py:20](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/io.py#L20); [src/mcp_atlassian/utils/decorators.py:136-142](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/decorators.py#L136-L142) (verified)
  - *To reach the next level:* Read-only mode defaults to off, and no confirmation must come from an authenticated principal.
- **B L2:** Most default writes are reversible (issue and page edits keep history), but issue and attachment deletion are permanent and comments, issues and requests notify people immediately. Evidence: searched `rg -n 'async def delete_'` in `src/mcp_atlassian/servers/` → 3 hits (delete_issue, delete_page and delete_attachment are exposed in the default toolsets) (verified)
  - *To reach the next level:* No checkpoint or preview for external actions, and deletes are irreversible.
- **Cap:** none

### C3 Tool & action scoping: 0.40 (high confidence)

Tools are narrow, purpose-built Jira and Confluence operations (create issue, add comment, update page) with typed arguments, issue-key patterns and bounded page sizes on most list tools, rather than a generic HTTP or query passthrough for writes. The few tools that read local files (attachment upload and page content from a file) resolve the path, following symlinks, and refuse anything outside the server's working directory. Search tools pass the model's JQL or CQL to Atlassian unchanged, and the issue search accepts any result limit. In the scored setup every toolset is enabled, including all write tools, and optional project and space filters are off, so a misused tool can act anywhere the user's account reaches.

- **S L2:** Narrow typed tools with resolved-path containment (resolve then is_relative_to) for every local file argument and page-size bounds on most list tools, but search tools pass raw JQL/CQL through and the issue search limit has no upper bound. Evidence: [src/mcp_atlassian/utils/io.py:46-56](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/io.py#L46-L56); [src/mcp_atlassian/confluence/attachments.py:152](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/confluence/attachments.py#L152); [src/mcp_atlassian/servers/confluence.py:175](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/servers/confluence.py#L175) (verified)
  - *To reach the next level:* Not every argument is bounded in code: query strings pass through unchanged and the issue-search limit is unbounded.
- **C L2:** Every tool has a pydantic schema, but bounds are applied per tool: some list tools cap limit at 50 while the issue search has no upper bound and query strings pass through. Evidence: [src/mcp_atlassian/servers/jira.py:860-863](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/servers/jira.py#L860-L863); [src/mcp_atlassian/servers/jira.py:1182-1185](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/servers/jira.py#L1182-L1185) (verified)
  - *To reach the next level:* No central policy layer applying bounds and scoping to every tool.
- **D L1:** With TOOLSETS unset all 25 toolsets load (a deprecation warning says the default will narrow later), so write tools are on; ENABLED_TOOLS, TOOLSETS and READ_ONLY_MODE can disable them. Evidence: [src/mcp_atlassian/utils/toolsets.py:195-203](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/toolsets.py#L195-L203) (verified)
  - *To reach the next level:* The default tool set is not read-only and not limited to the six core toolsets.
- **B L1:** Write tools reach every project and space the user's account can, and batch issue creation takes an unbounded JSON array. Evidence: [src/mcp_atlassian/servers/jira.py:1918-1925](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/servers/jira.py#L1918-L1925); [src/mcp_atlassian/utils/io.py:43-44](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/io.py#L43-L44) (verified)
  - *To reach the next level:* No default project/space scoping or quantity bounds on writes.
- **Cap:** none

### C4 Code-execution isolation: 1.00 (high confidence)

The server has no code-execution path: it makes HTTP calls to Atlassian and converts content between Markdown and Atlassian formats, and nothing in the source runs a shell, spawns a process, or evaluates model-supplied code. JQL and CQL strings are sent to Atlassian's search APIs, which interpret them server-side as read-only queries.

- **Structural absence:** searched `rg -n -S -e 'subprocess|os\.system|os\.popen|shell=True|\beval\(|\bexec\(|pickle|create_subprocess'` in `src/` → 0 hits (no process spawning, eval/exec or deserialization in the package source)

### C5 Untrusted input blast radius: 0.25 (high confidence)

Issue descriptions, comments, Confluence pages and service-desk requests are written by many people, including external customers, and the server returns them to the model as JSON fields alongside metadata, with no marker saying which text is untrusted. Tool descriptions and server instructions are plain and contain no directives aimed at the model. In the scored setup the same session can read that content, read private data across the user's Jira and Confluence, and write, comment, delete or upload local files, and nothing in the server breaks that combination; the read-only mode that would remove the write leg is off by default. A successful injection can therefore leak private data into a page or comment and take irreversible actions without any human step the server enforces.

- **S L2:** Tools return structured JSON that separates content fields from metadata (for example get_issue serialises a simplified issue dict). Evidence: [src/mcp_atlassian/servers/jira.py:794](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/servers/jira.py#L794); [src/mcp_atlassian/servers/jira.py:826](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/servers/jira.py#L826) (verified)
  - *To reach the next level:* No provenance or untrusted flag on returned content that a host could act on.
- **C L1:** The structured format applies across tools, but no source (issue bodies, comments, pages, customer requests) is distinguished as untrusted. Evidence: [src/mcp_atlassian/servers/jira.py:53](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/servers/jira.py#L53); [src/mcp_atlassian/servers/confluence.py:570](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/servers/confluence.py#L570) (verified)
  - *To reach the next level:* Untrusted text from other users is not marked on any tool output.
- **D L2:** The structured output is always on, while the read-only mode that drops the write leg is an operator setting that defaults to off. Evidence: [src/mcp_atlassian/utils/io.py:20](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/io.py#L20) (verified)
  - *To reach the next level:* The leg-dropping mode is not on by default.
- **B L0:** With all toolsets on, a hijacked session can copy private data into pages, comments or new issues (or upload local files under the working directory) and delete issues, with no server-enforced human step. Evidence: [src/mcp_atlassian/utils/toolsets.py:195-203](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/toolsets.py#L195-L203); [src/mcp_atlassian/servers/jira.py:3877](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/servers/jira.py#L3877); searched `rg -n 'async def delete_'` in `src/mcp_atlassian/servers/` → 3 hits (delete_issue, delete_page and delete_attachment are exposed in the default toolsets) (verified)
  - *To reach the next level:* No default mode separating untrusted reading from writes and egress.
- **Cap:** C5-WORSTCASE: In the default configuration a hijacked session can both leak private Jira/Confluence data and take irreversible actions with no server-enforced human step.

### C6 Memory, context & configuration integrity: 1.00 (high confidence)

The server keeps no memory between calls and loads no instruction files. Its only auto-loaded configuration is a .env file, which python-dotenv looks for starting from the installed package's own directory (for the README's uvx install, a user cache path) rather than the current project, and the server's tools cannot write local files (attachment downloads are returned inline). Nothing the model reads can therefore persist into later sessions through the server.

- **Structural absence:** searched `rg -n -S -e 'vector_store|save_memory|remember|AGENTS\.md|CLAUDE\.md|\.cursorrules|mcpServers'` in `src/` → 1 hits (the one hit is a sample host config printed by the OAuth setup wizard; no memory store or instruction-file loading); searched `rg -n -S -e 'load_dotenv|dotenv_values|find_dotenv'` in `src/` → 5 hits (import, a comment, dotenv_values() and two load_dotenv calls at startup; with no path, python-dotenv searches upward from the package's install directory, not the working directory); searched `rg -n -S -e 'open\(.*["'"'"']w|write_text|write_bytes'` in `src/` → 4 hits (OAuth token file in ~/.mcp-atlassian, a temporary-directory file during Markdown conversion, and two library download helpers that no tool calls; tools return attachments as base64)

### C7 Third-party extensions: 1.00 (high confidence)

The server loads no plugins, MCP servers, downloaded tools or model files at runtime. The only dynamic import is an opt-in OAuth-proxy storage backend whose module path the operator sets in an environment variable, which is operator configuration rather than an extension the model or content can add.

- **Structural absence:** searched `rg -n -S -e 'import_module|entry_points|load_plugin|trust_remote_code|stdio_client|pip install|npx'` in `src/` → 2 hits (both hits are the operator-configured OAuth client-storage factory import in servers/client_storage.py (opt-in proxy mode only))

### C8 Secrets & sensitive-data protection: 0.45 (high confidence)

Credentials come from environment variables (or the MCP host's config file), OAuth tokens are kept in the OS keyring with a 0600 file fallback, and the code masks tokens and authorization headers wherever it logs them. There is no telemetry or crash reporting. Credentials never enter tool results, but Jira and Confluence content returned to the model is not scanned for secrets, and error text from failed calls is passed back to the model as is. The default API token is long-lived and covers the user's whole Jira and Confluence access.

- **S L2:** A mask_sensitive helper and header masking are used on the main log paths, and OAuth tokens go to the OS keyring or a 0600 file. Evidence: [src/mcp_atlassian/utils/logging.py:80](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/logging.py#L80); [src/mcp_atlassian/jira/client.py:107](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/jira/client.py#L107); [src/mcp_atlassian/utils/oauth.py:373](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/oauth.py#L373); [src/mcp_atlassian/utils/oauth.py:420](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/oauth.py#L420) (verified)
  - *To reach the next level:* No redaction before model-bound messages, and the default API token sits in plaintext host config or environment rather than a secret store.
- **C L2:** Logs are masked and there is no subprocess or telemetry path, but tool error text is forwarded to the model unfiltered and returned content is not scanned. Evidence: [src/mcp_atlassian/utils/decorators.py:107-110](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/decorators.py#L107-L110) (verified)
  - *To reach the next level:* Model-bound results and error messages are not redacted.
- **D L2:** No telemetry exists and logging defaults to WARNING on stderr. Evidence: searched `rg -n -S -i -e 'sentry|posthog|telemetry|opentelemetry|mixpanel|amplitude'` in `src/` → 8 hits (all hits are the TimeInStatusEntry model name; no telemetry SDK); [src/mcp_atlassian/__init__.py:281-286](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/__init__.py#L281-L286) (verified)
  - *To reach the next level:* Verbose modes (-vv, MCP_VERY_VERBOSE) enable DEBUG on the root logger, including third-party HTTP libraries whose output is not passed through the masking helpers.
- **B L1:** A leaked default credential is a long-lived user API token or PAT, though the model never sees it. Evidence: [src/mcp_atlassian/jira/client.py:152-156](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/jira/client.py#L152-L156) (verified)
  - *To reach the next level:* Default credentials are neither narrowly scoped nor short-lived.
- **Cap:** none

### C9 Audit & traceability: 0.12 (high confidence)

The server keeps no audit record of what it did. Logs go to standard error at WARNING level by default, so successful tool calls (creates, updates, deletes) are not recorded at all; only failed calls and warnings appear, as unstructured text. Raising verbosity adds informational lines for some operations, but there is no structured per-call record, no actor attribution and no durable storage; the MCP host's own logs are not credited.

- **S L0:** At the default WARNING level no successful tool call is logged; per-operation lines exist only at INFO/DEBUG. Evidence: [src/mcp_atlassian/__init__.py:54](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/__init__.py#L54); searched `rg -n -S -i -e 'audit_log|auditlog|log_tool_call|tool_call_log'` in `src/` → 0 hits (no audit facility) (verified)
  - *To reach the next level:* No structured record of every tool call with arguments, result and timestamp.
- **C L1:** Every tool's failures are logged through the shared error decorator, so errors (not successes) are covered. Evidence: [src/mcp_atlassian/utils/decorators.py:107-108](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/decorators.py#L107-L108) (verified)
  - *To reach the next level:* Successful calls, including writes, are not recorded.
- **D L1:** Warnings and errors go to stderr by default, captured by whatever host launched the server. Evidence: [src/mcp_atlassian/utils/logging.py:35-38](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/logging.py#L35-L38) (verified)
  - *To reach the next level:* No record written by a component outside the server process, and per-call logging is opt-in via verbosity.
- **B L0:** Logging is best-effort stream output; actions proceed regardless and nothing is flushed to durable storage per action. Evidence: [src/mcp_atlassian/utils/logging.py:35-38](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/logging.py#L35-L38) (verified)
  - *To reach the next level:* No durable per-action record or surfaced logging failures.
- **Cap:** none

### C10 Limits & kill switch: 0.40 (high confidence)

The server bounds some of its own work: every Atlassian HTTP call has a 75-second timeout by default, inline attachments are capped at 50 MB, and most list tools cap page size at 50. Retries, a concurrency cap, an outbound rate limit, a circuit breaker and a pagination ceiling all exist but are off by default. The issue search accepts any limit and pages until it is reached, and attachments are read fully into memory before the size check. When the host process exits, the stdio server notices and cancels its main task, though synchronous HTTP calls already in flight run to their timeout.

- **S L2:** Server-enforced caps on some operations: per-request HTTP timeout, a 50 MB inline attachment cap and page-size bounds on most list tools. Evidence: [src/mcp_atlassian/jira/config.py:188](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/jira/config.py#L188); [src/mcp_atlassian/confluence/config.py:53](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/confluence/config.py#L53); [src/mcp_atlassian/utils/media.py:12](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/media.py#L12) (verified)
  - *To reach the next level:* No cap on every operation and no concurrency or rate limit by default.
- **C L2:** The HTTP timeout applies to every Jira and Confluence client, so every tool call is time-bounded per request. Evidence: [src/mcp_atlassian/jira/client.py:109-115](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/jira/client.py#L109-L115); [src/mcp_atlassian/jira/attachments.py:101-108](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/jira/attachments.py#L101-L108) (verified)
  - *To reach the next level:* Paginated loops and in-memory attachment reads are not bounded as a whole.
- **D L1:** Defaults exist, but the model can raise the issue-search limit at will and the pagination ceiling, rate limit and concurrency cap default to disabled. Evidence: [src/mcp_atlassian/servers/jira.py:860-863](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/servers/jira.py#L860-L863); [src/mcp_atlassian/utils/pagination.py:29-31](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/pagination.py#L29-L31); [src/mcp_atlassian/utils/http.py:349](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/http.py#L349); [src/mcp_atlassian/utils/http.py:283](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/utils/http.py#L283) (verified)
  - *To reach the next level:* The model can choose unbounded result sizes; no hard ceiling applies by default.
- **B L1:** A runaway caller can issue unlimited writes and large searches, limited only by Atlassian's own rate limits; on host exit the server task is cancelled but in-flight requests finish. Evidence: [src/mcp_atlassian/__init__.py:98-100](https://github.com/sooperset/mcp-atlassian/blob/0a5d2427144d72f5aade07d4fabd385421cec401/src/mcp_atlassian/__init__.py#L98-L100) (verified)
  - *To reach the next level:* No rate limit on side-effecting tools and stopping does not cancel in-flight HTTP calls.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Issue descriptions, comments, Confluence pages and service-desk requests by other users and customers, returned as JSON (src/mcp_atlassian/servers/jira.py:794) · [B] sensitive data/systems: Everything the user's API token can read in Jira and Confluence, plus local files under the working directory via attachment upload (src/mcp_atlassian/jira/client.py:155, src/mcp_atlassian/confluence/attachments.py:152) · [C] state change / egress: 40 write tools on by default, including issue/page/attachment deletion, comments and page creation (src/mcp_atlassian/servers/jira.py:2481, src/mcp_atlassian/utils/toolsets.py:203) · Same default session? Yes

## Highest-impact improvements
1. Ship TOOLSETS defaulting to the core toolsets and READ_ONLY_MODE on, requiring an explicit flag to expose write tools. (C5 B L0→L2, +0.100 before caps; Playbook 1)
2. Log every tool call (name, arguments, result status, timestamp) as structured records at the default log level, or to a dedicated audit file. (C9 S L0→L2, +0.150 before caps; Playbook 1 step 3)
3. Default to the core toolsets so write-heavy toolsets are opt-in, as the deprecation warning already plans. (C3 D L1→L2, +0.050 before caps; Playbook 3)
4. Mark returned issue, comment and page bodies with provenance (author, source) and an untrusted flag the host can act on. (C5 S L2→L3, +0.075 before caps; Playbook 1)
5. Enable the pagination ceiling and an outbound rate limit by default and bound the issue-search limit in its schema. (C10 D L1→L2, +0.050 before caps; Playbook 3 step 3)

## Re-audit log
- C3 S: L3 → L2. Path containment is solid, but the L3 anchor also requires bounds on arguments: search tools pass raw JQL/CQL and the issue-search limit (servers/jira.py:860-863) has no upper bound.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the stdio transport from the README Quick Start; the opt-in SSE/streamable-HTTP multi-user mode (per-user header tokens, OAuth proxy, unauthenticated-request rejection) was reviewed only at its entry points and not scored.
- Behaviour of third-party libraries (atlassian-python-api, FastMCP, python-dotenv) is inferred from how this code calls them; their source was not reviewed at the pinned versions.
- The .env lookup was judged user-scope for the README's uvx install; installing the package into a project's own virtual environment would make that project's .env reachable.
- Local-file containment uses the server's working directory as its root, which the MCP host chooses; what that directory holds was not assessed.
- No text aimed at AI reviewers was found in the repository.
