# Defense-in-Depth Score: Vane (Perplexica)

**Repo:** https://github.com/ItzCrazyKns/Vane · **Commit:** `348feca3e378fb4157b217724ed508dc707f853f` · **Reviewed:** 2026-10-05
**What it is:** Self-hosted AI answering engine (formerly Perplexica) that searches the web through a bundled SearxNG, reads pages and uploaded files, and writes cited answers with a choice of LLM providers.
**Category:** AI Assistants
**Scored configuration:** The README's recommended single Docker image (docker run -p 3000:3000 itzcrazykns1337/vane:latest, or the shipped docker-compose.yaml) with bundled SearxNG, provider keys entered in the setup screen, and no further hardening.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 3.2 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L0 | L0 | L2 | 0.17 | — | **0.17** | High |
| C2 | Approval gates | L0 | L0 | L0 | L2 | 0.10 | — | **0.10** | High |
| C3 | Tool & action scoping | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C4 | Code-execution isolation | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L0 | L2 | 0.25 | — | **0.25** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L0 | L1 | L0 | 0.12 | — | **0.12** | High |
| C9 | Audit & traceability | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |

Controls where a risk surface exists: 2.15 / 9.0 (24%); 1 criterion scored SA (surface absent).

A search-and-answer agent with a small tool set: it can search, fetch web pages and read uploaded files, but it cannot write files, run commands or send messages. Around that core there are almost no safeguards: the web app has no authentication (the README lists it as upcoming), so anyone who can reach the port can use the agent, read every chat and change provider settings and keys. Web content enters the model unmarked while a URL-fetch tool can reach any address, and the whole app runs as root in one container with the provider keys.

## Critical gaps
- A hijacked session can send data out through the always-on URL fetch with no gate, and with no user separation it can reach other users' chats. (ASI01, LLM01; C5). Evidence: [src/lib/scraper.ts:70-73](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/scraper.ts#L70-L73); [src/app/api/chats/route.ts:5](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/app/api/chats/route.ts#L5)
- Model-influenced execution (headless browser, math evaluator) shares a root process with the provider keys and all chat data, with no dedicated isolation. (ASI05; C4). Evidence: [Dockerfile:64](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/Dockerfile#L64); [src/lib/agents/search/widgets/calculationWidget.ts:58](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/widgets/calculationWidget.ts#L58)

## Criterion details

### C1 Identity & least privilege — 0.17 (high)

The agent acts with the deployment's own authority: the LLM provider keys entered at setup, and the network position of the server, which its URL-fetch tool can use to reach any host, including internal ones. There is no user identity at all, because the web app and its API have no authentication, so every request is treated the same and anyone who can reach the port can change providers and settings. The shipped image runs the app as root. A misused instance can spend the provider accounts and read from the network the server sits on, but it holds no cloud or source-control credentials.

- **S L1:** The agent uses operator-supplied provider keys from the shared config file, and its URL tool acts from the server's network position; no narrower identity exists. Evidence: [src/lib/models/registry.ts:20-28](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/models/registry.ts#L20-L28); [src/lib/scraper.ts:70-73](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/scraper.ts#L70-L73) (verified)
  - *To reach the next level:* No role-scoped identity: one set of provider keys and the server's full network reach serve every request.
- **C L0:** No API route checks who is calling; the only restriction on the URL tool is a sentence in its description telling the model not to call it unprompted. Evidence: searched `rg -n -i 'auth|login|password'` in `src/app/api` → 0 hits (No authentication or authorization in any API route.); [src/lib/agents/search/researcher/actions/scrapeURL.ts:64](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/researcher/actions/scrapeURL.ts#L64) (verified)
  - *To reach the next level:* No authorization check in code on any tool or API path.
- **D L0:** The default image runs the app as root and publishes it with no login; authentication is listed as an upcoming feature. Evidence: [Dockerfile:64](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/Dockerfile#L64); [docker-compose.yaml:7](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/docker-compose.yaml#L7); [README.md:229](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/README.md#L229) (verified)
  - *To reach the next level:* No reduced-privilege default; least privilege needs manual hardening (reverse proxy auth, non-root user).
- **B L2:** Misuse reaches the configured LLM provider accounts (spend) and read access to whatever the server's network can reach; there are no cloud, source-control or messaging credentials. Evidence: [src/lib/config/index.ts:8-11](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/config/index.ts#L8-L11); [src/lib/scraper.ts:70-73](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/scraper.ts#L70-L73) (verified)
  - *To reach the next level:* Provider keys are long-lived and not limited to the agent's read-only job.
- **Cap:** none

### C2 Approval gates — 0.10 (high)

There is no approval step anywhere, and nothing marks any tool as needing one. The agent's tools only search, fetch pages and read uploaded files, so it cannot write files, send messages or change data directly. Fetching a model-chosen URL does load that page in a full browser from the server's network position, which can have side effects on services that act on page loads, and nothing previews or confirms those fetches.

- **S L0:** No approval mechanism exists; the only confirm dialog in the code is the user's own chat-deletion prompt. Evidence: searched `rg -n -i 'approv|confirm'` in `src` → 7 hits (All hits are the chat-deletion confirmation dialog in DeleteChat.tsx, not tool gating.) (verified)
  - *To reach the next level:* No per-call human approval for any tool, including the URL fetch.
- **C L0:** Every tool call the model emits is executed directly by the action registry. Evidence: [src/lib/agents/search/researcher/index.ts:164-171](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/researcher/index.ts#L164-L171) (verified)
  - *To reach the next level:* The URL-fetch tool, the most powerful path, is not gated.
- **D L0:** There is no approval setting to turn on. Evidence: [src/lib/agents/search/researcher/actions/scrapeURL.ts:66](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/researcher/actions/scrapeURL.ts#L66) (verified)
  - *To reach the next level:* No approval on by default.
- **B L2:** No tool writes or deletes anything; the residual risk is page loads with side effects on reachable services, which cannot be undone and have no preview. Evidence: [src/lib/agents/search/researcher/actions/index.ts:10-16](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/researcher/actions/index.ts#L10-L16) (verified)
  - *To reach the next level:* No preview or dry-run for fetches and no rate limit on them.
- **Cap:** none

### C3 Tool & action scoping — 0.30 (high)

The tools are narrow by design: web, academic and social search through SearxNG, a search over uploaded files, and a URL fetch. Each caps how many queries or URLs one call can use (three), but nothing else is checked: the declared argument schemas are sent to the model and not enforced, and the URL fetch accepts any address, scheme or host, with no allowlist and no block on internal or local addresses. Which tools are offered depends on the chosen sources and mode, but the URL fetch is always on.

- **S L1:** Arguments are passed through except for a count cap; URLs are handed to the browser with no validation of scheme, host or address range. Evidence: [src/lib/agents/search/researcher/actions/scrapeURL.ts:68](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/researcher/actions/scrapeURL.ts#L68); [src/lib/agents/search/researcher/actions/scrapeURL.ts:50](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/researcher/actions/scrapeURL.ts#L50) (verified)
  - *To reach the next level:* No allowlist validation in code: URL scheme/host checks and internal-address blocking are missing.
- **C L1:** The only check on any tool is the same count cap (web, academic, social, uploads and URL tools each slice to three). Evidence: [src/lib/agents/search/researcher/actions/search/webSearch.ts:88-90](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/researcher/actions/search/webSearch.ts#L88-L90); [src/lib/agents/search/researcher/actions/registry.ts:73-79](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/researcher/actions/registry.ts#L73-L79) (verified)
  - *To reach the next level:* No shared validation layer; tool arguments are not parsed against their schemas.
- **D L2:** Tools are selected per request from sources and mode and none writes or executes, but the URL fetch is always offered. Evidence: [src/lib/agents/search/researcher/actions/scrapeURL.ts:66](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/researcher/actions/scrapeURL.ts#L66); [src/lib/agents/search/researcher/actions/search/webSearch.ts:84-86](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/researcher/actions/search/webSearch.ts#L84-L86) (verified)
  - *To reach the next level:* The arbitrary-URL fetch cannot be turned off and is not limited to explicit user requests in code.
- **B L1:** A misused fetch reaches any host the server can, including internal services and local addresses, limited only to three URLs per call. Evidence: [src/lib/scraper.ts:70-73](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/scraper.ts#L70-L73) (verified)
  - *To reach the next level:* No scoping of the fetch to public hosts and no per-session quantity bound.
- **Cap:** none

### C4 Code-execution isolation — 0.25 (high)

The agent never runs model-written programs, but two paths interpret text it does not control: the calculation widget evaluates a model-extracted expression with the mathjs expression parser inside the server process, and the URL fetch loads model-chosen pages, running their JavaScript, in a headless Chromium. Neither has dedicated isolation. The only separation is the recommended Docker container, a stock image that runs as root, gives the bundled search service's account passwordless sudo, and holds the provider keys and all chat data; non-Docker installs run everything directly on the host.

- **S L1:** Expressions are evaluated in-process by mathjs, whose safety rests on its own parser restrictions; the browser has no isolation of its own beyond the app container. Evidence: [src/lib/agents/search/widgets/calculationWidget.ts:58](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/widgets/calculationWidget.ts#L58); [src/lib/scraper.ts:16-27](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/scraper.ts#L16-L27) (verified)
  - *To reach the next level:* No OS-level boundary between executed content and the app process that holds the keys.
- **C L1:** Both interpretation paths, the math evaluator and the browser, run alongside the app with no separate boundary. Evidence: [src/lib/agents/search/widgets/calculationWidget.ts:4](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/widgets/calculationWidget.ts#L4); [src/lib/agents/search/researcher/actions/search/baseSearch.ts:368](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/researcher/actions/search/baseSearch.ts#L368) (verified)
  - *To reach the next level:* Neither execution path is separated from the app that holds credentials.
- **D L2:** The container is the README's recommended deployment, but the documented non-Docker install runs the same paths directly on the host. Evidence: [README.md:216](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/README.md#L216); [Dockerfile:77](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/Dockerfile#L77) (verified)
  - *To reach the next level:* No isolation that is always on and cannot be skipped by deployment choice.
- **B L0:** A failure lands in a root process (or as the host user) holding the provider keys, all chats and uploads, with unrestricted network; the bundled SearxNG account has passwordless sudo. Evidence: [Dockerfile:64](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/Dockerfile#L64); [Dockerfile:71](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/Dockerfile#L71); [src/lib/config/index.ts:8-11](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/config/index.ts#L8-L11) (verified)
  - *To reach the next level:* No non-root user, no secrets kept out of the executing process, no egress restriction.
- **Cap:** none

### C5 Untrusted input blast radius — 0.25 (high)

Web search results, fetched pages and uploaded documents go into the model's context wrapped in plain result tags, and tool results go back in as ordinary tool messages; nothing marks them as untrusted or changes what the agent may do after reading them. The only limit on the URL-fetch tool is a sentence in its description. A hijacked session can therefore send data out unattended by fetching an attacker URL with data in it, and because the instance has no users or authentication, it can also reach other people's chats through the app's own local API. It cannot take destructive actions.

- **S L1:** Untrusted content is only delimited with result tags and the URL tool carries a prompt instruction; nothing acts on provenance. Evidence: [src/lib/agents/search/index.ts:109](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/index.ts#L109); searched `rg -n -i 'untrusted|injection'` in `src` → 0 hits (No taint or injection handling anywhere.) (verified)
  - *To reach the next level:* No code that disables or gates egress once untrusted content is read.
- **C L1:** Only the final writer context wraps sources; tool results in the research loop enter as raw JSON tool messages. Evidence: [src/lib/agents/search/researcher/index.ts:175-182](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/researcher/index.ts#L175-L182) (verified)
  - *To reach the next level:* Tool results, fetched pages and uploads are not distinguished in the research loop.
- **D L2:** The delimiters are always applied and not configurable, but they are the only measure. Evidence: [src/lib/agents/search/index.ts:120](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/index.ts#L120) (verified)
  - *To reach the next level:* No structural limit that is on by default.
- **B L0:** A hijack can exfiltrate unattended via the URL tool, and with no user separation it can reach every user's chats through the app's local API (multi-user rule lowers L1 to L0). Evidence: [src/lib/scraper.ts:70-73](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/scraper.ts#L70-L73); [src/app/api/chats/route.ts:5](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/app/api/chats/route.ts#L5) (verified)
  - *To reach the next level:* Egress after untrusted input is not gated and the instance has no per-user separation.
- **Cap:** C5-WORSTCASE: With no user separation, a hijacked session can send other users' data out unattended through the URL-fetch tool.

### C6 Memory, context & configuration integrity — 0.25 (high)

The agent has no long-term memory tool and loads no instruction files from a workspace. What persists is chat history in SQLite, which the browser sends back as conversation history on follow-ups, and uploaded files with their embeddings. Neither is validated or tagged as untrusted, and both are shared by everyone using the instance: chats are listed globally and an uploaded file is found by its ID alone. Poisoned text in a chat persists for that conversation and can steer later tool use there, but chats are visible in the library and can be deleted.

- **S L1:** Model answers, including anything injected, are stored and replayed as assistant turns with no validation or provenance. Evidence: [src/lib/agents/search/index.ts:176-188](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/index.ts#L176-L188); [src/lib/hooks/useChat.tsx:203-215](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/hooks/useChat.tsx#L203-L215) (verified)
  - *To reach the next level:* Stored history is not presented as data or validated before reuse.
- **C L1:** No persistence path has a control; chat history and upload chunks are both reused as-is. Evidence: [src/lib/uploads/manager.ts:72-87](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/uploads/manager.ts#L72-L87) (verified)
  - *To reach the next level:* Neither chat history nor uploaded-file stores are controlled.
- **D L0:** All chats and uploads are shared across everyone who can reach the instance; there are no user namespaces. Evidence: [src/app/api/chats/route.ts:5](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/app/api/chats/route.ts#L5); [src/lib/uploads/manager.ts:66-70](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/uploads/manager.ts#L66-L70) (verified)
  - *To reach the next level:* No per-user isolation of chats or uploads.
- **B L2:** Poisoned history persists within one chat and can influence later tool calls there; it is visible in the library and deletable. Evidence: [src/app/api/chats/[id]/route.ts:55-56](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/app/api/chats/[id]/route.ts#L55-L56) (verified)
  - *To reach the next level:* Poisoned content is neither session-scoped nor reviewed before reuse.
- **Cap:** none

### C7 Third-party extensions — 1.00 (high)

Vane loads no plugins, MCP servers or third-party tools at runtime: its model providers and tools are a fixed list in the source. The optional local embedding provider downloads ONNX model weights from Hugging Face, which are data rather than code. The Docker build clones SearxNG unpinned, but that is a build-time dependency rather than an extension the agent loads.

- **Structural absence:** searched `rg -n -i 'mcp|plugin|trust_remote_code|npx'` in `src` → 0 hits (No MCP client, plugin loader or remote-code model loading.); searched `rg -n 'child_process|spawn\(|eval\(|new Function'` in `src` → 0 hits (No subprocess launch or dynamic code loading; providers are a fixed map in source.)

### C8 Secrets & sensitive-data protection — 0.12 (high)

Provider API keys are typed into the setup screen and stored in plaintext in a JSON config file in the data volume. There is no redaction helper anywhere, and with no authentication on the instance, anyone who can reach it can open the provider settings. The settings form masks key fields on screen and API errors return a generic message, but nothing else protects keys on server paths. There is no telemetry. A leaked key is long-lived and stays valid until the operator rotates it at the provider.

- **S L1:** Keys sit in a plaintext JSON file; the only masking is the password input type in the settings UI. Evidence: [src/lib/config/index.ts:128-133](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/config/index.ts#L128-L133); [src/lib/models/providers/openai/index.ts:111](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/models/providers/openai/index.ts#L111); searched `rg -n -i 'redact|mask'` in `src` → 0 hits (No redaction code.) (verified)
  - *To reach the next level:* No type-level masking or log filtering on server paths.
- **C L0:** No redaction exists on any server path; the generic error replies are not a secrets control. Evidence: [src/app/api/chats/route.ts:10-13](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/app/api/chats/route.ts#L10-L13); searched `rg -n -i 'redact|mask'` in `src` → 0 hits (No redaction code.) (verified)
  - *To reach the next level:* No path is redacted.
- **D L1:** No telemetry ships, but nothing is redacted by default and the settings are open to any client. Evidence: searched `rg -n -i 'posthog|sentry|telemetry'` in `src` → 0 hits (No telemetry SDK.) (verified)
  - *To reach the next level:* No redaction on by default.
- **B L0:** Long-lived provider keys, billed to the operator, are stored in the root-run app process that any client can drive, with no short-lived or scoped alternative. Evidence: [src/lib/config/index.ts:195-199](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/config/index.ts#L195-L199) (verified)
  - *To reach the next level:* Keys are long-lived and reachable from the unauthenticated app.
- **Cap:** none

### C9 Audit & traceability — 0.25 (high)

Each answer is saved in SQLite with its response blocks, which include the research steps (search queries issued and pages read) and sources. That gives a partial, after-the-fact record, but the full blocks are only written when an answer completes, tool arguments and timings are not recorded as such, there is no actor or user attribution, and any client can delete a chat and its record through the open API. Everything else is console logging of errors.

- **S L1:** Response blocks capture research sub-steps (queries, URLs read) per message, but not a structured per-call record with arguments and timestamps. Evidence: [src/lib/agents/search/index.ts:176-188](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/index.ts#L176-L188) (verified)
  - *To reach the next level:* No structured record of every tool call with arguments, status and timestamps.
- **C L1:** Only the chat UI path persists; the /api/search developer API uses random chat IDs and its runs are not tied to any listed chat. Evidence: [src/app/api/search/route.ts:53-67](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/app/api/search/route.ts#L53-L67) (verified)
  - *To reach the next level:* Not every entry point and tool call is recorded.
- **D L1:** Records are on by default but any client can delete them through the unauthenticated chat API. Evidence: [src/app/api/chats/[id]/route.ts:55-56](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/app/api/chats/[id]/route.ts#L55-L56) (verified)
  - *To reach the next level:* Records are not protected from deletion by the same interface that drives the agent.
- **B L1:** The query row is written up front but research blocks are flushed only at the end of the answer, so a crash loses the trajectory. Evidence: [src/lib/agents/search/index.ts:22-31](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/index.ts#L22-L31) (verified)
  - *To reach the next level:* Records are not flushed per action.
- **Cap:** none

### C10 Limits & kill switch — 0.45 (high)

The research loop has a hard iteration cap of 2, 6 or 25 rounds depending on the mode the request chooses, SearxNG queries time out after 10 seconds and page loads after 20. LLM calls have no timeout, there is no token or cost cap, and nothing rate-limits requests to the unauthenticated API. Stopping does not stop much: when the client disconnects only the response stream closes, and the agent keeps searching, fetching and calling the model until it finishes.

- **S L2:** Iteration cap plus timeouts on search and page loads, enforced in code. Evidence: [src/lib/agents/search/researcher/index.ts:15-20](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/researcher/index.ts#L15-L20); [src/lib/searxng.ts:42](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/searxng.ts#L42); [src/lib/scraper.ts:8](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/scraper.ts#L8) (verified)
  - *To reach the next level:* No token or cost cap, no LLM-call timeout, no rate limit.
- **C L2:** The cap covers the research loop and timeouts cover search and fetch; classifier, widget, extractor and writer model calls are unbounded in time. Evidence: [src/lib/agents/search/researcher/index.ts:59](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/agents/search/researcher/index.ts#L59) (verified)
  - *To reach the next level:* Model calls and parallel tool calls per round are not bounded.
- **D L2:** Defaults are sensible and the model cannot raise them, but any client picks the 25-round quality mode and there is no ceiling on request volume. Evidence: [src/app/api/chat/route.ts:36-38](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/app/api/chat/route.ts#L36-L38) (verified)
  - *To reach the next level:* No operator ceiling on mode or request rate for unauthenticated clients.
- **B L1:** Client abort only closes the stream; the agent run continues to completion, and the session object lingers for 30 minutes. Evidence: [src/app/api/chat/route.ts:235-238](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/app/api/chat/route.ts#L235-L238); [src/lib/session.ts:17](https://github.com/ItzCrazyKns/Vane/blob/348feca3e378fb4157b217724ed508dc707f853f/src/lib/session.ts#L17) (verified)
  - *To reach the next level:* No cancellation of the running agent or in-flight calls.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Web results, fetched pages and uploads enter context unmarked (src/lib/agents/search/index.ts:109, src/lib/agents/search/researcher/index.ts:180) · [B] sensitive data/systems: Shared chat history and uploaded documents for every user of the instance (src/app/api/chats/route.ts:5, src/lib/uploads/manager.ts:66) · [C] state change / egress: URL fetch to any model-chosen address (src/lib/scraper.ts:70) · Same default session? Yes

## Highest-impact improvements
1. Validate URLs in scrape_url: http(s) only, resolve and block private, loopback and link-local addresses, and recheck after redirects. — C3 S L1→L3, +0.150 before caps (Playbook 3)
2. Ship authentication (even a single admin password) and per-user chat and upload namespaces. — C1 C L0→L2, +0.150 before caps (Playbook 4)
3. Run the app as a non-root user, drop the sudoers entry, and move the headless browser into its own hardened container. — C4 B L0→L2, +0.100 before caps (Playbook 3)
4. Disable the URL-fetch tool once search results have been read unless the user's own message contains the URL. — C5 S L1→L3, +0.150 before caps (Playbook 1)
5. Cancel the agent run on client disconnect and add timeouts and a token budget to every model call. — C10 B L1→L2, +0.050 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, built or probed.
- The pinned commit is 5 commits after tag v1.12.2 and carries no tag of its own; package.json reports 1.12.2.
- Behaviour of third-party libraries (mathjs expression parser, Playwright/Chromium, Hugging Face transformers.js) was inferred from their documented behaviour, not read at the installed versions.
- The bundled SearxNG is cloned unpinned at image build time and was not reviewed; only Vane's own code and the shipped Docker files were.
- The slim image (external SearxNG) and non-Docker installs were not scored separately; they share the same application code.
- No reviewer-steering text was found in the repository.
