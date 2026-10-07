# Defense-in-Depth Score: Local Deep Researcher

**Repo:** https://github.com/langchain-ai/local-deep-researcher · **Commit:** `ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d` · **Reviewed:** 2026-10-05
**What it is:** Local web research assistant built on LangGraph that loops search, summarise and reflect steps with a local Ollama or LM Studio model and returns a cited markdown report.
**Category:** AI Assistants
**Scored configuration:** README quickstart: `langgraph dev` local server (127.0.0.1:2024) with default Configuration values (DuckDuckGo search, full-page fetch on, local Ollama, 3 research loops) and no .env overrides.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials opt-in · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 7.1 / 10.0 (Strong)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L3 | L3 | 0.60 | — | **0.60** | High |
| C2 | Approval gates | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C3 | Tool & action scoping | L2 | L2 | L3 | L2 | 0.55 | — | **0.55** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L1 | L2 | L2 | L3 | 0.47 | — | **0.47** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L2 | L2 | L3 | 0.47 | — | **0.47** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | Low |
| C10 | Limits & kill switch | L2 | L2 | L3 | L2 | 0.55 | — | **0.55** | High |

Controls where a risk surface exists: 3.05 / 6.0 (51%); 4 criteria scored SA (surface absent).

A small, fixed research pipeline: the local model only writes search queries and summaries, and the code decides everything else, so there is no code execution, no file access, no messaging and, by default, no credentials. The main exposure is web content: pages go to the model with only cosmetic tags, can steer later search queries, and the full-page downloader fetches any result URL without address restrictions. There is no audit trail of its own, no secret redaction, and no time or cost budget beyond a three-loop cap.

## Critical gaps
- None: no criterion is capped and no blast radius is at the worst level.

## Criterion details

### C1 Identity & least privilege — 0.60 (high)

The researcher holds no cloud, repository or account credentials. In the default setup it searches DuckDuckGo, which needs no key at all, and talks to a local Ollama model; if the operator switches to Tavily or Perplexity, it uses that service's API key from the environment and nothing else. There are no subprocesses that could inherit the environment and no per-request authorization layer, but there is also very little authority to misuse: a stolen key can only spend search credits.

- **S L2:** Optional search-provider keys (TAVILY_API_KEY via TavilyClient, PERPLEXITY_API_KEY as a bearer header) are the only credentials; each is used only for its own provider's search API — [src/ollama_deep_researcher/utils.py:303](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/utils.py#L303); [src/ollama_deep_researcher/utils.py:337-341](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/utils.py#L337-L341) (verified)
  - *To reach the next level:* No per-capability or short-lived credential scoping; a configured key is long-lived and static for the whole process.
- **C L2:** Every outbound call goes through the fixed search helpers or the local LLM client; no tool constructs other privileged clients and nothing is spawned as a subprocess — [src/ollama_deep_researcher/graph.py:213-256](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/graph.py#L213-L256); searched `rg -n 'subprocess|os\.system|\beval\(|\bexec\(|Popen|pickle|importlib|__import__'` in `src Dockerfile` → 0 hits (No subprocess or dynamic code paths that could inherit credentials.) (verified)
  - *To reach the next level:* No authorization check per request or per caller; whoever can start a run uses the configured keys.
- **D L3:** The default search API is DuckDuckGo and the default LLM is a local Ollama endpoint, so a fresh install holds no external credentials; paid providers need an explicit operator setting — [src/ollama_deep_researcher/configuration.py:34-36](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/configuration.py#L34-L36); [src/ollama_deep_researcher/configuration.py:72-75](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/configuration.py#L72-L75) (verified)
  - *To reach the next level:* Per-run configuration from the LangGraph API can switch the search provider when no environment value pins it, and there is no time-bound elevation.
- **B L3:** A misused or stolen key reaches only one search provider's account (query spend and history); there is no write access to any other system — [src/ollama_deep_researcher/utils.py:354-356](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/utils.py#L354-L356) (verified)
  - *To reach the next level:* Keys are long-lived and not task-scoped or revocable per run.
- **Cap:** none
- **Notes:** The scored mode is the README's local `langgraph dev` server. The Docker image's network access configuration is not locked down; it is not the scored mode.

### C2 Approval gates — 1.00 (high)

The researcher has no consequential actions: it only runs web searches, downloads search-result pages and writes a summary into its own run state. It never writes files, sends messages, posts content or calls a write API, so there is nothing for an approval gate to protect. Paid search calls (Tavily, Perplexity, when configured) cost credits, which is bounded by the loop limit scored in C10.

- **Structural absence:** searched `rg -n 'open\(|\.write\(|unlink|shutil|os\.remove|smtp|sendmail'` in `src` → 0 hits (No file writes, deletions or mail sending anywhere.); searched `rg -n 'requests\.(post|put|delete|patch)|\.post\(|\.put\(|\.delete\('` in `src` → 1 hits (The single hit is the Perplexity search request (a query, not a state change).)
- **Notes:** Credit spend on paid search providers is the only side effect; it is covered under C10.

### C3 Tool & action scoping — 0.55 (high)

The model never gets a general-purpose tool. The workflow is fixed in code: the model only writes the text of the next search query, and the code decides which search service to call and how many results to take (one to three). When full-page fetching is on, which it is by default, the code downloads every result URL with a 10-second timeout and trims the text to about 4,000 characters, but it applies no host or address restrictions to those URLs. Nothing can write, execute or send.

- **S L2:** Narrow, code-fixed operations: the model supplies only a query string, result counts are fixed in code, and fetched page text is truncated; the full-page fetcher takes whatever URL the search returns without host or address checks — [src/ollama_deep_researcher/utils.py:141-162](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/utils.py#L141-L162); [src/ollama_deep_researcher/graph.py:213-218](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/graph.py#L213-L218); [src/ollama_deep_researcher/utils.py:116-117](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/utils.py#L116-L117) (verified)
  - *To reach the next level:* No URL allowlist or block on internal and link-local addresses for the full-page fetcher.
- **C L2:** Every search path uses code-fixed result counts and the shared truncation; the page fetch used by DuckDuckGo and SearXNG has no destination validation — [src/ollama_deep_researcher/utils.py:203-204](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/utils.py#L203-L204); [src/ollama_deep_researcher/utils.py:263-264](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/utils.py#L263-L264) (verified)
  - *To reach the next level:* Validation does not cover fetched destinations on every path.
- **D L3:** The only capabilities are search and page download; there are no write, exec or messaging tools, and the model cannot add tools because the graph is compiled from a fixed node list — [src/ollama_deep_researcher/graph.py:451-465](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/graph.py#L451-L465) (verified)
  - *To reach the next level:* No per-task narrowing beyond the fixed graph (L4 requires both a read-only set and per-task allowlists; D is also limited to one level above S).
- **B L2:** Misuse is read-only and count-bounded (at most three results per loop), but full-page fetching can reach any host the machine can reach — [src/ollama_deep_researcher/configuration.py:37-41](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/configuration.py#L37-L41); [src/ollama_deep_researcher/utils.py:156-157](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/utils.py#L156-L157) (verified)
  - *To reach the next level:* Fetch destinations are not restricted to public internet hosts.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

The researcher never runs model-generated or downloaded text as code. Model output is only parsed as JSON to extract a query string, and downloaded pages are converted to markdown text. There is no shell, eval, subprocess or script execution anywhere in its source.

- **Structural absence:** searched `rg -n 'subprocess|os\.system|\beval\(|\bexec\(|Popen|pickle|importlib|__import__'` in `src Dockerfile` → 0 hits (No code-execution path in the agent source or container entrypoint.)

### C5 Untrusted input blast radius — 0.47 (high)

Web pages and search snippets are the researcher's whole input, and they go straight to the model. The code wraps them in simple context tags but nothing acts on those tags. What limits a hijack is the fixed workflow: injected text can change the summary and steer the next search query, but it cannot pick tools, write anything, or reach secrets, because the model never sees credentials and has no actions beyond searching. The remaining outbound channel is the text of search queries sent to the configured search provider.

- **S L1:** Untrusted web text is wrapped in <Context>/<New Context> tags with no enforcement; the fixed graph limits the model to choosing query text, but that text, steered by page content, still drives further searches and fetches — [src/ollama_deep_researcher/graph.py:287-297](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/graph.py#L287-L297); [src/ollama_deep_researcher/graph.py:372-374](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/graph.py#L372-L374) (verified)
  - *To reach the next level:* No rule-of-two enforcement: after untrusted content is read, outbound search continues with model-chosen query text.
- **C L2:** Web results, the only untrusted source, are delimited on every summarisation call; the running summary that carries them into the reflection step is only separated by plain === markers — [src/ollama_deep_researcher/graph.py:372-374](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/graph.py#L372-L374) (verified)
  - *To reach the next level:* Derived content (the running summary) is not marked as untrusted when it re-enters the reflection prompt.
- **D L2:** The delimiting is fixed in code and cannot be switched off by content or configuration — [src/ollama_deep_researcher/graph.py:287-297](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/graph.py#L287-L297) (verified)
  - *To reach the next level:* D is limited to one level above S; a stronger structural control would be needed to earn more.
- **B L3:** A hijacked run can only distort the summary and leak the research topic or summary through search queries to the configured provider; no credentials are in context and there is no state-changing action — [src/ollama_deep_researcher/graph.py:372-374](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/graph.py#L372-L374); [src/ollama_deep_researcher/utils.py:188-191](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/utils.py#L188-L191) (verified)
  - *To reach the next level:* Search queries remain an outbound channel for session content.
- **Cap:** none

### C6 Memory, context & configuration integrity — 1.00 (high)

Nothing the researcher reads persists into future behaviour. It has no memory store, vector index or instruction files, and it does not load configuration from any workspace it reads. Run state (sources and the running summary) lives in the LangGraph thread for that run. Configuration comes from the operator's environment, the project's own .env, or the run settings in LangGraph Studio.

- **Structural absence:** searched `rg -n -i 'memory|store|checkpoint|vector|save|open\('` in `src` → 0 hits (No persistence layer, memory store or file reads in the agent.); searched `rg -n -i 'load_dotenv|AGENTS\.md|CLAUDE\.md|plugin|mcp|entry_points|trust_remote_code'` in `src langgraph.json` → 0 hits (No auto-loaded instruction or workspace files; the .env referenced by langgraph.json is the project's own operator file.)
- **Notes:** Reusing a LangGraph thread carries its previous running summary into the next run; that is session state for one thread, not cross-session memory.

### C7 Third-party extensions — 1.00 (high)

The researcher loads no third-party code at runtime: there are no plugins, MCP servers, downloaded tools or model files that execute code. Models are served by a separate local Ollama or LM Studio process. The quickstart and Docker image launch the LangGraph CLI with `uvx --refresh`, which is the project's own unpinned dependency rather than an extension.

- **Structural absence:** searched `rg -n -i 'load_dotenv|AGENTS\.md|CLAUDE\.md|plugin|mcp|entry_points|trust_remote_code'` in `src langgraph.json` → 0 hits (No plugin, MCP or remote-code loading.); searched `rg -n 'subprocess|os\.system|\beval\(|\bexec\(|Popen|pickle|importlib|__import__'` in `src Dockerfile` → 0 hits (No dynamic imports or deserialisation of downloaded files.)
- **Notes:** The `uvx --refresh` launch of langgraph-cli fetches the latest release at start; this is general dependency hygiene and outside C7's scope.

### C8 Secrets & sensitive-data protection — 0.47 (high)

API keys come from environment variables and are never placed in prompts, so the model never sees them. There is no redaction or masking code at all: full model responses are printed to the console, and when LangSmith tracing is enabled by the operator, search queries and results are sent to LangSmith. The default setup with DuckDuckGo holds no secret, and any optional key is scoped to one search service.

- **S L1:** Secrets are read from environment variables and attached only in provider request code; there is no masking, redaction or secret type anywhere — [src/ollama_deep_researcher/utils.py:337-341](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/utils.py#L337-L341); searched `rg -n -i 'redact|mask|SecretStr'` in `src` → 0 hits (No redaction or masking code.) (verified)
  - *To reach the next level:* No type-level masking or log filters on any path.
- **C L2:** By design keys stay out of model-bound messages, console prints and traced function arguments, since traced helpers take only the query and flags — [src/ollama_deep_researcher/utils.py:277-280](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/utils.py#L277-L280); [src/ollama_deep_researcher/graph.py:83](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/graph.py#L83) (verified)
  - *To reach the next level:* No protection on error output or opt-in tracing payloads beyond keeping keys out of them by construction.
- **D L2:** Tracing is opt-in (the @traceable decorator only reports when LangSmith tracing is enabled); full model output and page-fetch errors are printed to stdout by default — [src/ollama_deep_researcher/utils.py:165](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/utils.py#L165); [src/ollama_deep_researcher/graph.py:83](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/graph.py#L83) (verified)
  - *To reach the next level:* Console output includes full model responses by default.
- **B L3:** The default DuckDuckGo search needs no key; optional Tavily or Perplexity keys are scoped to one search service and rotatable by the operator — [src/ollama_deep_researcher/configuration.py:34-36](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/configuration.py#L34-L36) (verified)
  - *To reach the next level:* Optional keys are long-lived rather than per-task.
- **Cap:** none

### C9 Audit & traceability — 0.40 (low)

The researcher has no audit log of its own. Its graph state accumulates every search's formatted results and source list, and the LangGraph runtime keeps that state per thread, which lets you see what was searched and fetched in a run. Beyond that, there are console prints and opt-in LangSmith tracing. Nothing records who started a run, and the dev server's storage is not built for durability or tamper resistance.

- **S L2:** Run state appends each search's results and source URLs (operator.add reducers), giving a structured per-step record; timestamps and step history are inferred from the LangGraph runtime's checkpointing — [src/ollama_deep_researcher/state.py:7-12](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/state.py#L7-L12) (inferred)
  - *To reach the next level:* No actor attribution (who started the run) or correlation identifiers.
- **C L2:** Every search and fetch flows through the web_research node, whose results and sources are appended to state — [src/ollama_deep_researcher/graph.py:258-262](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/graph.py#L258-L262) (verified)
  - *To reach the next level:* Configuration changes per run are not recorded alongside the run.
- **D L1:** The record exists by default but lives in the in-memory LangGraph dev runtime inside the agent's own process — [langgraph.json:1-11](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/langgraph.json#L1-L11) (inferred)
  - *To reach the next level:* Records are not written by a component outside the agent's process.
- **B L1:** Persistence is best-effort in the dev runtime; failures to record are not surfaced by the agent — [langgraph.json:1-11](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/langgraph.json#L1-L11) (inferred)
  - *To reach the next level:* No per-action durable flush or replay guarantee.
- **Cap:** none

### C10 Limits & kill switch — 0.55 (high)

Research depth is capped in code at three loops by default (four searches in total), and the model cannot change that number; only the operator or the run configuration can. Full-page downloads have a 10-second timeout, but calls to the local model and the Perplexity API have none. There is no token or cost budget and no ceiling on the configurable loop count. Stopping a run uses the LangGraph server's cooperative cancellation.

- **S L2:** An iteration cap (max_web_research_loops, enforced in the routing function) plus a 10-second per-fetch timeout — [src/ollama_deep_researcher/graph.py:437-441](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/graph.py#L437-L441); [src/ollama_deep_researcher/utils.py:156](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/utils.py#L156) (verified)
  - *To reach the next level:* No token or cost cap, and model calls and the Perplexity request have no timeouts.
- **C L2:** The loop cap bounds the whole graph and page fetches carry a timeout; there are no sub-agents or background tasks — [src/ollama_deep_researcher/utils.py:354-356](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/utils.py#L354-L356) (verified)
  - *To reach the next level:* Timeouts do not cover every kind of step (LLM calls, Perplexity search).
- **D L3:** A sensible default of 3 loops; the model has no way to raise it because it is read only from environment or run configuration — [src/ollama_deep_researcher/configuration.py:19-23](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/configuration.py#L19-L23) (verified)
  - *To reach the next level:* No hard ceiling that configuration cannot exceed.
- **B L2:** With default settings a runaway is bounded to a handful of searches, but a hung model or Perplexity call can block the run indefinitely — [src/ollama_deep_researcher/utils.py:354-356](https://github.com/langchain-ai/local-deep-researcher/blob/ba505ffc3d7e897bc48b6169fa2cb1ec859dd38d/src/ollama_deep_researcher/utils.py#L354-L356) (verified)
  - *To reach the next level:* No per-run wall-clock or spend ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Search snippets and downloaded pages enter the summariser prompt (src/ollama_deep_researcher/graph.py:287-297) · [B] sensitive data/systems: Only the user's research topic and run summary; no credentials in context (src/ollama_deep_researcher/graph.py:372-374) · [C] state change / egress: Model-written search queries go to the search provider and result URLs are fetched (src/ollama_deep_researcher/utils.py:141-162) · Same default session? Yes

## Highest-impact improvements
1. Restrict full-page fetching to public hosts (block loopback, private and link-local ranges, recheck redirects). — C3 S L2→L3, +0.075 before caps (Playbook 3)
2. Add timeouts to every model and search call and a per-run wall-clock and token budget. — C10 S L2→L3, +0.075 before caps (Playbook 3 step 3)
3. Write a structured per-step run log (query, URLs fetched, model, caller) outside the dev server's memory. — C9 D L1→L2, +0.050 before caps (Playbook 1 step 3)
4. Stop printing full model responses by default and route diagnostics through a logger with redaction. — C8 S L1→L2, +0.075 before caps (Playbook 4)
5. Mark the running summary as derived from untrusted web content when it re-enters the reflection prompt. — C5 C L2→L3, +0.075 before caps (Playbook 1)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Behaviour of the LangGraph dev server (langgraph-cli / langgraph-api: checkpoint persistence and cancellation) and of LangGraph Studio's hosted UI was inferred from general library behaviour, not read in those dependencies.
- The Docker deployment mode was reviewed but not scored; the score covers the README's local `langgraph dev` quickstart.
- Behaviour of the duckduckgo-search, tavily-python and SearxSearchWrapper libraries (redirect handling, timeouts) was not examined.
- The repository's GitHub Actions workflows (Claude Code review) are maintainer CI, not part of the agent, and were not scored.
- No reviewer-steering text was found in the repository.
