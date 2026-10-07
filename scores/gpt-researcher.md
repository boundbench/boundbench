# Defense-in-Depth Score: GPT Researcher

**Repo:** https://github.com/assafelovic/gpt-researcher · **Commit:** `0957c301ed06c2a5857b834358c7227c739041d4` (v3.7.0) · **Reviewed:** 2026-10-03
**What it is:** Autonomous deep-research agent over web and local data; also MCP
**Category:** AI Assistants
**Scored configuration:** The FastAPI/WebSocket research server started as the README leads (uvicorn main:app; main.py, the Dockerfile and docker-compose bind 0.0.0.0) with the default config: Tavily search, BeautifulSoup scraper, report_source=web, no auth, MCP off unless a request supplies mcp_configs.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication no

## Score: 2.2 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L2 | 0.10 | — | **0.10** | High |
| C3 | Tool & action scoping | L2 | L2 | L3 | L2 | 0.55 | — | **0.55** | High |
| C4 | Code-execution isolation | L2 | L2 | L0 | L0 | 0.30 | G1 | **0.30** (alt) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L2 | 0.10 | — | **0.10** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L1 | 0.05 | C7-RCELOAD | **0.05** | Medium |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L1 | L2 | L2 | L2 | 0.42 | — | **0.42** | High |
| C10 | Limits & kill switch | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |


GPT Researcher's research pipeline itself is mostly read-only: it searches the web, fetches pages through an SSRF filter, and writes a report. The dominant risk is the server around it. No endpoint authenticates callers, and a research request may include an MCP server 'command' that the server spawns as a process. So anyone who can reach the port can run commands with the user's account and API keys (root in the shipped docker-compose). Run it only on localhost behind your own authentication, and treat stored reports and chat history as shared and untrusted.

## Critical gaps
- Any unauthenticated client of /ws can make the server spawn an arbitrary MCP stdio command as the server's OS user (root under docker-compose), with the operator's keys reachable. (ASI03, T3, ASI05; C1) — [backend/server/app.py:408-412](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/app.py#L408-L412); [backend/server/server_utils.py:420-422](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/server_utils.py#L420-L422); [gpt_researcher/mcp/client.py:95-103](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/client.py#L95-L103); [docker-compose.yml:21](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/docker-compose.yml#L21)
- Request-supplied MCP commands run as unsandboxed host subprocesses with access to the user's account, network and .env secrets. (ASI05, T11; C4) — [gpt_researcher/mcp/client.py:95-103](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/client.py#L95-L103); [gpt_researcher/mcp/client.py:143](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/client.py#L143); [main.py:29](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/main.py#L29)
- By default the server launches any MCP server command (e.g. 'npx -y <pkg>') supplied by any unauthenticated WebSocket client, with no operator consent or allowlist. (ASI04, T17; C7) — [backend/server/server_utils.py:420-422](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/server_utils.py#L420-L422); [backend/server/websocket_manager.py:174](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/websocket_manager.py#L174); [gpt_researcher/mcp/client.py:95-103](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/client.py#L95-L103); [SECURITY.md:39-41](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/SECURITY.md#L39-L41)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

The server holds the operator's API keys and runs as the operator's OS user (root inside the shipped docker-compose service), and none of its REST or WebSocket endpoints authenticate the caller. Any client that can reach the port acts with the server's full authority: it can spend the keys, read and delete every stored report, and supply an MCP 'command' that the server launches as a process. The maintainers' SECURITY.md states this is by design and leaves authentication to the operator. Nothing narrows the authority the agent inherits.

- **S L0:** No identity or scoping of its own: the process uses the operator's ambient OS user and long-lived provider keys loaded from the environment/.env, and lends that whole authority to any caller. — [main.py:29](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/main.py#L29); [docker-compose.yml:21](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/docker-compose.yml#L21); [gpt_researcher/mcp/client.py:95-103](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/client.py#L95-L103) (verified)
  - *To reach the next level:* A dedicated, scoped identity (e.g., per-capability keys, no arbitrary process launch) would be needed for L1+.
- **C L0:** No authorization layer exists on any route; the WebSocket accepts every connection without checking credentials. — [backend/server/app.py:408-412](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/app.py#L408-L412); [backend/server/websocket_manager.py:56](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/websocket_manager.py#L56); searched `rg -n -i "Depends\(|HTTPBearer|APIKeyHeader|HTTPBasic|OAuth2|verify_token"` in `backend main.py` → 0 hits (No authentication or authorization dependency on any REST or WebSocket route.) (verified)
  - *To reach the next level:* An authorization check on at least the main research path (/ws) is needed for L1.
- **D L0:** Default deployments bind every interface (main.py, Dockerfile HOST=0.0.0.0, compose 8000:8000) and compose runs the service as root with keys in its environment. — [main.py:37](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/main.py#L37); [docker-compose.yml:21](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/docker-compose.yml#L21); [Dockerfile:41](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/Dockerfile#L41) (verified)
  - *To reach the next level:* A narrower default (localhost-only bind, non-root, auth on) would be needed.
- **B L0:** A caller can launch arbitrary commands via request-supplied MCP stdio configs, giving the user's entire OS account (or root in the compose container with all API keys). — [backend/server/server_utils.py:420-422](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/server_utils.py#L420-L422); [backend/server/websocket_manager.py:174](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/websocket_manager.py#L174); [gpt_researcher/mcp/client.py:95-103](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/client.py#L95-L103); [docker-compose.yml:21](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/docker-compose.yml#L21) (verified)
  - *To reach the next level:* Removing client-controlled process launch and limiting the process to read-only, single-purpose credentials would be needed.
- **Cap:** none
- **Notes:** SECURITY.md documents the unauthenticated design as intentional; the score reflects the shipped default regardless. Exposure of the local WebSocket beyond direct clients is not locked down.

### C2 Approval gates — 0.10 (high)

There is no approval step anywhere. The default research flow only searches and reads web pages, so it makes no consequential changes on its own. When a request enables MCP servers, the model picks MCP tools and arguments and they execute immediately with no human in the loop, whatever those tools do. A 'human_feedback' WebSocket command exists but is a stub that only prints the message.

- **S L0:** No approval mechanism; model-selected MCP tool calls run directly. — [gpt_researcher/mcp/research.py:100-113](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/research.py#L100-L113); [backend/server/server_utils.py:189-192](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/server_utils.py#L189-L192) (verified)
  - *To reach the next level:* Per-call human approval of tool calls would be needed for L2+.
- **C L0:** The most powerful action path (MCP tool execution) is not gated. — [gpt_researcher/mcp/research.py:100-113](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/research.py#L100-L113); searched `rg -n -i "approv|confirm|human_feedback"` in `gpt_researcher backend` → 5 hits (All 5 hits are the human_feedback stub handler and its dispatch; none gate a tool call.) (verified)
  - *To reach the next level:* At least flagged mutating tools would need to be gated.
- **D L0:** No approval exists to be on by default. — [gpt_researcher/mcp/research.py:100-113](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/research.py#L100-L113) (verified)
  - *To reach the next level:* Approval on by default for consequential tools would be needed.
- **B L2:** In the default request (no MCP) the model can only search and fetch pages; with request-supplied MCP servers it can invoke whatever write tools those servers expose, irreversibly, with no previews or rate limits. — [backend/server/websocket_manager.py:174](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/websocket_manager.py#L174); [gpt_researcher/config/variables/default.py:45](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/config/variables/default.py#L45); [gpt_researcher/mcp/research.py:100-113](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/research.py#L100-L113) (verified)
  - *To reach the next level:* Checkpoints/previews for consequential actions or a code-enforced read-only default for every connector would be needed.
- **Cap:** none

### C3 Tool & action scoping — 0.55 (high)

The default tools are narrow: search queries go to a configured search API and the scraper fetches result URLs after a shared check that allows only http(s) and rejects hosts resolving to private, loopback, link-local or metadata addresses. The code itself notes a DNS-rebinding window, and URL and transport handling does not cover every path. MCP tools receive raw model-chosen arguments, and MCP path restrictions are not a strict boundary. Write-capable tools only appear when a request adds MCP servers.

- **S L2:** validate_url enforces scheme plus public-IP resolution, but re-resolution leaves a TOCTOU window (acknowledged in code) and enforcement does not cover every path. — [gpt_researcher/utils/url_security.py:97-98](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/utils/url_security.py#L97-L98); [gpt_researcher/utils/url_security.py:22-24](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/utils/url_security.py#L22-L24) (verified)
  - *To reach the next level:* Pin the validated IP and harden enforcement to reach L3.
- **C L2:** The scraper and online document loader validate URLs; MCP tool arguments pass through unvalidated, and MCP path restrictions are not a strict boundary. — [gpt_researcher/scraper/scraper.py:202-205](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/scraper/scraper.py#L202-L205); [gpt_researcher/document/online_document.py:42](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/document/online_document.py#L42); [gpt_researcher/mcp/research.py:100-113](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/research.py#L100-L113) (verified)
  - *To reach the next level:* A shared validation layer wrapping extension (MCP) tools is needed for L3.
- **D L3:** Default tool set is search plus HTTP GET scraping; MCP (possibly write/exec) tools load only when the request sets mcp_enabled with configs, and the model cannot enable them. — [gpt_researcher/config/variables/default.py:28](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/config/variables/default.py#L28); [gpt_researcher/config/variables/default.py:45](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/config/variables/default.py#L45); [backend/server/websocket_manager.py:174](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/websocket_manager.py#L174) (verified)
  - *To reach the next level:* Per-task tool allowlists and removing client-level enabling of arbitrary MCP servers would be needed for L4.
- **B L2:** Default tools are read-only GETs but can reach any public host; MCP tools, when enabled, have whatever reach their server has. — [backend/report_type/basic_report/basic_report.py:62-64](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/report_type/basic_report/basic_report.py#L62-L64) (verified)
  - *To reach the next level:* Quantity bounds and host allowlists on fetches would be needed for L3.
- **Cap:** none

### C4 Code-execution isolation — 0.30 (high)

The project never runs model-written code, but it launches subprocesses from text it does not control: a research request may carry MCP server configs whose 'command' and 'args' the server spawns as a local stdio process with no isolation, as the same OS user. Package installs via pip are also triggered at runtime for some optional scrapers and LLM providers. The shipped Docker image runs as a non-root user, but the shipped docker-compose overrides that to root and passes the API keys into the container; Docker is an optional deployment, so it is scored as an opt-in mechanism.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Request-supplied MCP commands run as same-user host subprocesses with no sandbox. — [gpt_researcher/mcp/client.py:95-103](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/client.py#L95-L103); [gpt_researcher/mcp/client.py:143](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/client.py#L143); searched `rg -n -i "sandbox|seccomp|cap_drop|no-new-privileges|read_only"` in `gpt_researcher backend Dockerfile docker-compose.yml` → 1 hits (The single hit is Chrome's --no-sandbox flag in the optional browser scraper, i.e. the opposite of a control.) (verified)
    - *To reach the next level:* Any OS-level separation (container, low-privilege user) on the default path would be L2.
  - **C L0:** No execution path is isolated (MCP stdio launch, runtime pip install). — [gpt_researcher/mcp/client.py:95-103](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/client.py#L95-L103); [gpt_researcher/scraper/scraper.py:182-184](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/scraper/scraper.py#L182-L184); [gpt_researcher/llm_provider/generic/base.py:415](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/llm_provider/generic/base.py#L415) (verified)
    - *To reach the next level:* The main exec path would need to be sandboxed.
  - **D L0:** No isolation exists in the default local run. — [main.py:37](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/main.py#L37); [gpt_researcher/mcp/client.py:95-103](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/client.py#L95-L103) (verified)
    - *To reach the next level:* Isolation on by default would be needed.
  - **B L0:** A spawned command has the user's full host account, home directory, network and the .env file with keys. — [main.py:29](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/main.py#L29); [gpt_researcher/mcp/client.py:95-103](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/client.py#L95-L103); [backend/server/server_utils.py:420-422](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/server_utils.py#L420-L422) (verified)
    - *To reach the next level:* Workspace-only mount, no secrets and restricted egress would be needed.
- **opt-in Docker deployment (Dockerfile / docker-compose)** (alt; raw 0.30, cap G1 → 0.30) ← counted
  - **S L2:** The image is a stock container; the Dockerfile creates a non-root user but docker-compose sets user: root, with default capabilities and no seccomp/read-only hardening. — [Dockerfile:59](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/Dockerfile#L59); [docker-compose.yml:21](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/docker-compose.yml#L21) (verified)
    - *To reach the next level:* Non-root, dropped capabilities, no-new-privileges and a read-only root filesystem would be needed for L3.
  - **C L2:** Everything the server spawns runs inside the container when deployed this way. — [Dockerfile:64](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/Dockerfile#L64) (verified)
    - *To reach the next level:* Fail-closed coverage with a documented escape hatch would be needed for L3.
  - **D L0:** Docker is an optional deployment; the README leads with a host uvicorn run. — [README.md:131](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/README.md#L131) (verified)
    - *To reach the next level:* Running containerised by default would be needed.
  - **B L0:** The compose container receives the API keys in its environment, runs as root, has unrestricted network and read-write host mounts. — [docker-compose.yml:6-9](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/docker-compose.yml#L6-L9); [docker-compose.yml:21](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/docker-compose.yml#L21); [docker-compose.yml:17-20](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/docker-compose.yml#L17-L20) (verified)
    - *To reach the next level:* No secrets in the container and restricted egress would be needed.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.10 (high)

Scraped web pages, search results, uploaded documents and MCP tool results all go straight into the model's prompts with no separation from instructions, and there is no detection or provenance handling. In the default web-research configuration the model sees mostly public content plus the user's query, has no tools that change state, and its only outbound channels are search queries to the search provider and a further unattended egress path. When a request enables local documents or MCP servers, injected content can steer private data out or drive MCP tools with no human involved.

- **S L0:** Nothing limits a hijacked model; no delimiting, detection or taint tracking. — searched `rg -n -i "untrusted|prompt.injection|spotlight|quarantin"` in `gpt_researcher backend` → 1 hits (Only hit is a docstring in url_security.py about untrusted URLs, not prompt content.) (verified)
  - *To reach the next level:* Even detection/spotlighting would be L1.
- **C L0:** No untrusted source is distinguished. — [backend/chat/chat.py:234](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/chat/chat.py#L234) (verified)
  - *To reach the next level:* At least one source handled would be L1.
- **D L0:** No control to enable. — [backend/chat/chat.py:234](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/chat/chat.py#L234) (verified)
  - *To reach the next level:* A control on by default would be needed.
- **B L2:** Default sessions read untrusted web content but hold little sensitive data and take no state-changing actions; egress exists through search queries and a further unattended path, and request options (local docs, MCP) widen it. — [gpt_researcher/config/variables/default.py:33](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/config/variables/default.py#L33); [backend/chat/chat.py:106](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/chat/chat.py#L106) (verified)
  - *To reach the next level:* Removing outbound channels from sessions that read untrusted content would be needed for L3.
- **Cap:** none
- **Notes:** Rated on the default report_source=web flow; with report_source=local/hybrid or MCP enabled per request, B would drop toward L1/L0.

### C6 Memory, context & configuration integrity — 0.00 (high)

Generated reports and chat histories are kept in one global JSON store that any unauthenticated client can list, overwrite or delete. When someone chats about a stored report, the report text (model output derived from untrusted web pages) is placed into the system prompt and the stored chat history is replayed with whatever roles it contains, including 'system'. There is no per-user namespace, validation, provenance, or review of what gets persisted, so poisoned content persists across sessions and users and can drive the chat's web-search tool.

- **S L0:** Model-written reports and client-written chat messages persist unvalidated and are re-injected as trusted context (system prompt / arbitrary roles). — [backend/server/app.py:211-220](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/app.py#L211-L220); [backend/chat/chat.py:248-253](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/chat/chat.py#L248-L253); [backend/chat/chat.py:234](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/chat/chat.py#L234) (verified)
  - *To reach the next level:* Provenance on stored entries and presenting them as data would be needed for L2.
- **C L0:** Neither the report store nor uploaded documents in DOC_PATH have any controlled write path. — [backend/server/report_store.py:44-48](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/report_store.py#L44-L48); [backend/server/app.py:398-400](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/app.py#L398-L400) (verified)
  - *To reach the next level:* At least one store controlled would be L1.
- **D L0:** One store shared by every client; listing returns all reports. — [backend/server/app.py:183-187](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/app.py#L183-L187); searched `rg -n -i "user_id|namespace|tenant"` in `backend/server gpt_researcher/memory backend/chat` → 0 hits (No user or namespace scoping in the store or chat.) (verified)
  - *To reach the next level:* Per-user/session namespaces enforced in queries would be L2.
- **B L0:** Poisoned reports/chat persist across sessions and users and are replayed into a model that can call the quick_search tool. — [backend/server/app.py:287-295](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/app.py#L287-L295); [backend/chat/chat.py:106](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/chat/chat.py#L106) (verified)
  - *To reach the next level:* Session scoping or easy inspection/purge with review would be needed.
- **Cap:** none

### C7 Third-party extensions — 0.05 (medium)

MCP servers are the main extension type, and they come from the request rather than from operator configuration: a client sends a command, arguments and environment, and the server spawns it. Nothing pins, hashes or allowlists what runs, and there is no consent step beyond the requester's own toggle, which any unauthenticated client can set. Separately, some optional scrapers and LLM providers pip-install unpinned packages at runtime when missing, and retriever plugins are loaded by name from installed entry points. Launched MCP processes run as the same OS user and can read the .env file holding the operator's keys.

- **S L0:** Executes whatever command the request supplies, unpinned and unverified; runtime pip installs are unpinned. — [gpt_researcher/mcp/client.py:95-103](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/client.py#L95-L103); [gpt_researcher/llm_provider/generic/base.py:415](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/llm_provider/generic/base.py#L415) (verified)
  - *To reach the next level:* Pinned versions would be needed for L2.
- **C L0:** No extension type is verified. — [gpt_researcher/mcp/client.py:95-103](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/client.py#L95-L103); searched `rg -n -i "sha256|hashlib|signature|pinned"` in `gpt_researcher/mcp gpt_researcher/retrievers/mcp` → 0 hits (No integrity checks in the MCP path.) (verified)
  - *To reach the next level:* At least one verified extension type would be L1.
- **D L0:** Any client can add an MCP server per request with no operator consent or display of what will run. — [backend/server/server_utils.py:420-422](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/server_utils.py#L420-L422); [backend/server/websocket_manager.py:174](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/websocket_manager.py#L174); [SECURITY.md:39-41](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/SECURITY.md#L39-L41) (verified)
  - *To reach the next level:* Operator-scoped allowlisting with a display of the exact command would be needed for L2/L3.
- **B L1:** MCP servers run as separate same-user processes; the mcp library's stdio client passes a reduced default environment unless 'env' is given, but the process can read the .env file and the user's home directory. — [gpt_researcher/mcp/client.py:95-103](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/client.py#L95-L103); [main.py:29](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/main.py#L29) (inferred)
  - *To reach the next level:* Separate process with no access to the agent's credential files would be needed for L2.
- **Cap:** C7-RCELOAD — By default the server launches any MCP server command (e.g. 'npx -y <pkg>') supplied by any unauthenticated WebSocket client, with no operator consent or allowlist.

### C8 Secrets & sensitive-data protection — 0.25 (high)

API keys come from environment variables or a .env file and are only attached to provider HTTP calls, so they are not placed into model prompts. There is no redaction or masking anywhere. Research logs (queries, sources, context, report) are written to outputs/*.json and served over an unauthenticated /outputs static route, and the chat endpoint logs full model responses at INFO. LangSmith tracing turns on automatically in the multi-agent path whenever a LangChain key is present, sending prompts to a third party.

- **S L1:** Secrets come from env/.env; no masking or SecretStr wrapping anywhere. — [main.py:29](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/main.py#L29); searched `rg -n -i "redact|SecretStr|mask"` in `gpt_researcher backend` → 0 hits (No redaction helpers exist.) (verified)
  - *To reach the next level:* Type-level masking and log filters on main paths would be L2.
- **C L1:** Model-bound messages carry no keys by design; logs, the served outputs folder and subprocess-reachable .env are unprotected. — [backend/server/server_utils.py:47](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/server_utils.py#L47); [backend/server/app.py:75](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/app.py#L75) (verified)
  - *To reach the next level:* Logs and transcripts would also need protection for L2.
- **D L1:** Content logging to served files is on by default; LangSmith tracing auto-enables in the multi-agent path when a key is set. — [multi_agents/main.py:43-44](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/multi_agents/main.py#L43-L44); [backend/server/app.py:443](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/app.py#L443) (verified)
  - *To reach the next level:* Opt-in telemetry and reasonable logging defaults would be L2.
- **B L1:** Long-lived LLM/search keys, moderately scoped (billing), readable by any spawned MCP process via .env and spendable by any client. — [docker-compose.yml:6-9](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/docker-compose.yml#L6-L9); [main.py:29](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/main.py#L29) (verified)
  - *To reach the next level:* Scoped or short-lived keys would be needed for L2/L3.
- **Cap:** none

### C9 Audit & traceability — 0.42 (high)

Each research run writes a JSON log file of streamed events (queries, sources, progress) and the server logs to logs/app.log at INFO, so the main pipeline leaves an unstructured trail. MCP tool calls log only the tool name at INFO; their arguments are logged at debug level. Records carry no caller identity (there is no authentication), sit in the working directory, and the per-run logs are served publicly.

- **S L1:** Event logs are unstructured progress messages; MCP tool arguments are debug-only; no actor attribution. — [gpt_researcher/mcp/research.py:93-98](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/research.py#L93-L98); [backend/server/server_utils.py:75-80](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/server_utils.py#L75-L80) (verified)
  - *To reach the next level:* A structured record of every tool call with arguments and result status would be L2.
- **C L2:** Built-in research steps and chat searches are logged; MCP (extension) calls lack arguments. — [backend/chat/chat.py:105](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/chat/chat.py#L105); [gpt_researcher/mcp/research.py:93](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/mcp/research.py#L93) (verified)
  - *To reach the next level:* Extension and sub-agent calls with arguments would be needed for L3.
- **D L2:** On by default, written by the server process to logs/ and outputs/ (the model has no file tools). — [main.py:15](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/main.py#L15); [backend/server/server_utils.py:86-87](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/server_utils.py#L86-L87) (verified)
  - *To reach the next level:* Writing through a component the process cannot alter would be L3.
- **B L2:** The per-run JSON log is rewritten synchronously on every event and write errors propagate to the run. — [backend/server/server_utils.py:64-87](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/server_utils.py#L64-L87) (verified)
  - *To reach the next level:* A replayable durable trajectory would be needed for L3.
- **Cap:** none

### C10 Limits & kill switch — 0.38 (high)

The research pipeline is a fixed sequence of stages rather than an open loop, and outbound HTTP calls have per-request timeouts. But the number of sub-queries is only suggested to the model (MAX_ITERATIONS is passed into the prompt and not enforced), clients can raise max_search_results without bound, and there is no wall-clock or cost cap. Closing the WebSocket cancels the running asyncio task, while REST /report/ background jobs cannot be stopped and executor threads finish their work.

- **S L1:** Iteration count is advisory (prompt); per-call HTTP timeouts exist; no cost or wall-clock cap. — [gpt_researcher/actions/query_processing.py:113](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/actions/query_processing.py#L113); [gpt_researcher/actions/query_processing.py:156](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/actions/query_processing.py#L156); searched `rg -n -i "max_cost|budget|cost_limit"` in `gpt_researcher backend` → 3 hits (All 3 hits are comments about token budgets in context filtering, not spend limits.) (verified)
  - *To reach the next level:* An enforced step cap plus a run-level time or cost cap would be L2.
- **C L2:** Top-level pipeline plus tool (HTTP) timeouts; MCP retrieval has a 300 s wait. — [gpt_researcher/retrievers/mcp/retriever.py:286](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/retrievers/mcp/retriever.py#L286); [gpt_researcher/scraper/beautiful_soup/beautiful_soup.py:70](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/scraper/beautiful_soup/beautiful_soup.py#L70) (verified)
  - *To reach the next level:* Sub-agents/background tasks counting against one budget would be L3.
- **D L2:** Sensible defaults (MAX_ITERATIONS 3, 5 results/query) but any client can raise max_search_results. — [gpt_researcher/config/variables/default.py:26](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/gpt_researcher/config/variables/default.py#L26); [backend/report_type/basic_report/basic_report.py:62-64](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/report_type/basic_report/basic_report.py#L62-L64) (verified)
  - *To reach the next level:* Limits the requester cannot raise would be L3.
- **B L1:** No spend ceiling; REST background jobs cannot be cancelled and threads keep running after a WebSocket cancel. — [backend/server/app.py:375-376](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/app.py#L375-L376); [backend/server/server_utils.py:406-408](https://github.com/assafelovic/gpt-researcher/blob/0957c301ed06c2a5857b834358c7227c739041d4/backend/server/server_utils.py#L406-L408) (verified)
  - *To reach the next level:* Tight per-run ceilings and cancellation of pending calls would be needed.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Scraped web pages and search results (gpt_researcher/scraper/scraper.py:213-222), uploaded docs, MCP results · [B] sensitive data/systems: Uploaded local documents when report_source=local/hybrid (backend/server/app.py:138) and the shared report store (backend/server/app.py:135); API keys stay out of prompts · [C] state change / egress: Model-chosen search queries to the search API, chat quick_search (backend/chat/chat.py:106), a further unattended egress path, and MCP tool calls when enabled (gpt_researcher/mcp/research.py:109) · Same default session? Yes

## Highest-impact improvements
1. Stop accepting MCP stdio 'command'/'args' from requests; only launch servers from an operator-defined MCP_SERVERS allowlist and show the exact command. — C7 D L0→L3, +0.150 before caps (Playbook 3)
2. Add optional-but-default-on API-key auth to /ws and REST routes, and default to binding 127.0.0.1. — C1 C L0→L1, +0.075 before caps (Playbook 4)
3. Require per-call human approval (showing tool name and arguments) before executing MCP tool calls, or restrict to read-only-annotated tools. — C2 S L0→L3, +0.225 before caps (Playbook 5)
4. Namespace the report store per user/session and stop replaying client-supplied chat roles; present stored reports as quoted data. — C6 S L0→L2, +0.150 before caps (Playbook 2)
5. Truncate sub-queries to MAX_ITERATIONS in code and add a per-run wall-clock and cost cap. — C10 S L1→L2, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the FastAPI/WebSocket server the README leads with; the pip library (GPTResearcher class), the NextJS frontend, the separate gptr-mcp server (not in this repo), deep_agents/, and the multi_agents LangGraph/AG2 flows were reviewed only where the server reaches them.
- Behaviour of third-party libraries (the mcp SDK's default stdio environment) is inferred from library defaults, not verified in this repo.
- SECURITY.md declares unauthenticated access and request-supplied MCP commands to be by design and the operator's responsibility; scores reflect the shipped default regardless.
- No reviewer-injection attempts were found in the repository.
