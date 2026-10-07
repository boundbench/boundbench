# Defense-in-Depth Score: Mem0

**Repo:** https://github.com/mem0ai/mem0 · **Commit:** `abb81c88e1f738a8117d8293530fbc31a5ef8fd9` (v2.2.1) · **Reviewed:** 2026-10-04
**What it is:** Memory layer for AI agents: an LLM extracts facts from conversations into a vector store and serves them back on search.
**Category:** Agent Frameworks
**Scored configuration:** Python SDK `Memory()` with default MemoryConfig (OpenAI LLM and embedder, local Qdrant at /tmp/qdrant, SQLite history in ~/.mem0, telemetry on, infer=True).
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents no · external communication no

## Score: 3.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L2 | L2 | 0.35 | — | **0.35** | High |
| C2 | Approval gates | L0 | L0 | L0 | L3 | 0.15 | — | **0.15** | High |
| C3 | Tool & action scoping | L3 | L2 | L2 | L2 | 0.57 | — | **0.57** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C7 | Third-party extensions | L1 | L0 | L1 | L0 | 0.12 | — | **0.12** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |

Controls where a risk surface exists: 2.80 / 9.0 (31%); 1 criterion scored SA (surface absent).

Mem0's core is small: it runs no code, and its LLM can only add memories inside a scope fixed by code. The main risk is persistence. Anything said in a conversation, including injected instructions, is turned into lasting memory automatically, with no review and no untrusted label, and is served back to agents in later sessions. Multi-tenant apps should also know that get, update, delete and history by memory ID don't check which user owns the memory. Telemetry is on by default.

## Critical gaps
- Optional HuggingFace/sentence-transformers models and the spaCy model mem0 downloads itself load in-process with the application's full credentials and environment. (ASI04, T17, LLM03; C7) — [mem0/utils/spacy_models.py:30-35](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/utils/spacy_models.py#L30-L35); [mem0/embeddings/huggingface.py:27](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/embeddings/huggingface.py#L27)

## Criterion details

### C1 Identity & least privilege — 0.35 (high)

Mem0 has no identity or authorization layer of its own. Every operation runs with the single LLM key and the single vector-store credential the developer configures, and that store holds all users' memories in one collection, separated only by a user_id/agent_id/run_id payload filter that the caller supplies. Add, search and get_all require such a filter, and the LLM can't pick or widen its scope. But get, update, delete and history take only a memory ID and never check which user owns it. An app that passes memory IDs from user input therefore lets one user read or change another user's memories.

- **S L1:** One static LLM key and one vector-store credential serve every user and operation; tenant separation is a caller-supplied payload filter, not a scoped credential or principal check. — [mem0/llms/openai.py:50](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/llms/openai.py#L50); [mem0/memory/main.py:392-394](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L392-L394) (verified)
  - *To reach the next level:* No per-scope or per-request credential, and no authorization check tied to an authenticated principal.
- **C L1:** Scope filters are enforced on add/search/get_all/delete_all, but get, delete, update and history act on a bare memory_id with no ownership check. — [mem0/memory/main.py:1233](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L1233); [mem0/memory/main.py:1892](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L1892); [mem0/memory/storage.py:227-238](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/storage.py#L227-L238) (verified)
  - *To reach the next level:* By-ID operations (get/update/delete/history) do not verify the memory belongs to the requesting scope.
- **D L2:** By default every add/search must name a scope (raises otherwise), and identity keys in caller metadata are stripped so the scope can't be widened through metadata. — [mem0/memory/main.py:392-394](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L392-L394); [mem0/memory/main.py:364](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L364) (verified)
  - *To reach the next level:* Scope remains a caller-asserted string; nothing binds it to an authenticated user, and by-ID calls ignore it.
- **B L2:** If scoping fails, the holder can read, rewrite and delete every tenant's memories in the shared store and use the configured LLM key; no other system is reachable from the library. — [mem0/memory/main.py:1233](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L1233); [mem0/llms/openai.py:50](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/llms/openai.py#L50) (verified)
  - *To reach the next level:* Store credential is not narrowed to one tenant or to read-only.
- **Cap:** none

### C2 Approval gates — 0.15 (high)

The LLM's only state-changing action is writing extracted memories to the store during add(). That happens automatically, with no human review and no hook to approve or reject a memory before it is saved. The extraction pipeline is add-only: it never updates or deletes existing memories. Each write gets a returned ID and a history row, so a bad memory can be found and deleted afterwards. Mem0 itself takes no external actions such as sending messages.

- **S L0:** Extracted memories are inserted directly after JSON parsing; no approval step or review callback exists. — [mem0/memory/main.py:980](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L980); [mem0/memory/main.py:1017-1041](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L1017-L1041); searched `rg -n 'approv|confirm|human_in_the_loop|review' --type py` in `mem0` → 10 hits (All hits are prompt text or code comments (prompts.py, main.py comments, llms/base.py model list, opensearch comment); none is an approval gate.) (verified)
  - *To reach the next level:* No per-write approval or review hook for model-extracted memories.
- **C L0:** The single model-driven write path (inferred memory insertion) has no gate. — [mem0/memory/main.py:1017-1041](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L1017-L1041) (verified)
  - *To reach the next level:* No gate exists on the inferred-write path.
- **D L0:** No approval mechanism exists to enable. — searched `rg -n 'approv|confirm|human_in_the_loop|review' --type py` in `mem0` → 10 hits (All hits are prompt text or code comments (prompts.py, main.py comments, llms/base.py model list, opensearch comment); none is an approval gate.) (verified)
  - *To reach the next level:* No approval mode, even opt-in.
- **B L3:** Model-driven changes are add-only inserts within the caller's scope, each logged with an ID in history and removable via delete(); no external or destructive action is model-reachable. — [mem0/memory/main.py:1080-1086](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L1080-L1086); [mem0/memory/main.py:1017-1041](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L1017-L1041); [mem0/memory/storage.py:108-119](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/storage.py#L108-L119) (verified)
  - *To reach the next level:* No rate or quantity limit on how many memories one add() can insert, and no point-in-time rollback.
- **Cap:** none

### C3 Tool & action scoping — 0.57 (high)

The model gets no general tools. Its output is parsed as JSON, and only memory text and an attribution field are kept. Code then inserts that text as new memories under a scope that code, not the model, fixed. Query parameters are validated (threshold and top_k bounds, entity-ID checks), and vector-store queries use parameterized SQL. The model can't choose an operation, a target scope or a destination. However, nothing limits how many memories one call can create, and the developer-facing by-ID methods skip scope validation.

- **S L3:** The model's capability is reduced to emitting memory text; operation (ADD) and scope are fixed in code, numeric search bounds are validated, and stores use parameterized queries. — [mem0/memory/main.py:980](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L980); [mem0/memory/main.py:926](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L926); [mem0/memory/main.py:212-236](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L212-L236); [mem0/vector_stores/pgvector.py:414](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/vector_stores/pgvector.py#L414) (verified)
  - *To reach the next level:* No bound on the number or size of extracted memories per call, and no single central policy layer.
- **C L2:** Validation applies on add/search/get_all/delete_all; get/update/delete/history accept a bare memory_id without scope validation. — [mem0/memory/main.py:1233](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L1233); [mem0/memory/main.py:1892](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L1892) (verified)
  - *To reach the next level:* By-ID methods are not covered by scope validation.
- **D L2:** By default (infer=True) the model's single capability is a write (memory insert); there is no read-only mode for inference. — [mem0/memory/main.py:770](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L770) (verified)
  - *To reach the next level:* No read-only default; inferred writes are on unless the developer passes infer=False.
- **B L2:** A misused extraction can write arbitrary text into one scope's memory store, unbounded in count, but cannot touch other scopes or delete data. — [mem0/memory/main.py:1017-1041](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L1017-L1041); [mem0/memory/main.py:926](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L926) (verified)
  - *To reach the next level:* No quantity bounds on inserted memories.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

The core SDK never runs model-generated or workspace-supplied text as code. It has no shell, Python execution, eval, or template rendering, and the model's output is only parsed as JSON. The only subprocess in reach is spaCy's fixed model download, which runs only when the optional NLP extra is installed. It is scored under third-party extensions.

- **Structural absence:** searched `rg -n 'subprocess|os\.system|os\.popen|\bexec\(|\beval\('` in `mem0` → 1 hits (Only hit is mem0/reranker/huggingface_reranker.py:61 self.model.eval() (PyTorch eval mode), not code interpretation.); searched `rg -n 'trust_remote_code|torch\.load|pickle\.load'` in `mem0` → 0 hits (No remote-code model loading or raw pickle loads; FAISS legacy docstore uses a restricted SafeUnpickler.)

### C5 Untrusted input blast radius — 0.05 (high)

Conversation content of any origin goes into the extraction prompt as plain markdown sections, with no injection handling. The model's output is saved as durable memory without being marked as untrusted. Within mem0 itself, a hijacked extraction can only add memories to the current user's scope: it can't read other users' data, delete anything, or send data out. But mem0's own OpenAI-compatible proxy pastes stored memories into the user turn as plain text and forwards the caller's tools to the model. A one-time injection can therefore come back in later sessions and steer a host agent's tool calls.

- **S L0:** Untrusted messages are concatenated into the extraction prompt under markdown headers with no provenance handling, detection, or capability restriction. — [mem0/configs/prompts.py:1040](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/configs/prompts.py#L1040); [mem0/memory/main.py:922](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L922) (verified)
  - *To reach the next level:* No spotlighting, detection, or capability restriction once untrusted content is read.
- **C L0:** All message roles and the stored last-10 messages enter context with equal standing; returned memories carry no untrusted flag in the proxy injection. — [mem0/configs/prompts.py:1040](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/configs/prompts.py#L1040); [mem0/proxy/main.py:181-186](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/proxy/main.py#L181-L186) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished anywhere in the pipeline.
- **D L0:** No control exists to be on by default. — [mem0/configs/prompts.py:1040](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/configs/prompts.py#L1040) (verified)
  - *To reach the next level:* No untrusted-input control at all.
- **B L1:** mem0 itself has no egress or destructive path for the model (add-only, scope-pinned), but per the framework rule its proxy re-injects persisted attacker text as plain user-turn content alongside caller-supplied tools, so the host agent's tool use can be steered across sessions. — [mem0/proxy/main.py:181-186](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/proxy/main.py#L181-L186); [mem0/proxy/main.py:124](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/proxy/main.py#L124); [mem0/memory/main.py:1080-1086](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L1080-L1086) (verified)
  - *To reach the next level:* Re-served memories are not tagged untrusted, and nothing stops them from driving host tool calls.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.30 (high)

Persistent memory is mem0's core purpose, and the model writes to it automatically. Facts extracted from any conversation are stored with no validation, review or expiry by default (expiration is opt-in). Each write is recorded in a SQLite history table. Memories are separated by a required user, agent or run scope that the model can't change. However, the default local store sits at /tmp/qdrant, a shared system directory, and nothing tags stored memories as untrusted when they are served back. A poisoned memory persists across the user's sessions until someone deletes it.

- **S L1:** Model-extracted text is written to durable memory unvalidated; writes are logged in the history table but not gated, and are re-served as plain context. — [mem0/memory/main.py:1017-1041](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L1017-L1041); [mem0/memory/storage.py:108-119](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/storage.py#L108-L119); [mem0/proxy/main.py:181-186](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/proxy/main.py#L181-L186) (verified)
  - *To reach the next level:* No provenance-as-data presentation or write validation/gating.
- **C L1:** The history log covers the main vector store; the entity store and raw messages table (re-read as last-10 context) have no equivalent control. — [mem0/memory/main.py:922](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L922); [mem0/memory/main.py:1207](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L1207) (verified)
  - *To reach the next level:* Entity store and messages table are uncontrolled.
- **D L2:** Scope is mandatory and enforced in queries; identity keys can't be set via metadata and the model can't pick a scope. — [mem0/memory/main.py:392-394](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L392-L394); [mem0/memory/main.py:926](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L926); [mem0/configs/vector_stores/qdrant.py:16](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/configs/vector_stores/qdrant.py#L16) (verified)
  - *To reach the next level:* Default store path is the shared /tmp/qdrant and retention is unlimited by default; stronger isolation is capped by the weak write control.
- **B L1:** Poisoned memories persist across the user's sessions and, via the proxy or host agent, can influence tool use; get_all/delete allow purging after the fact. — [mem0/proxy/main.py:181-186](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/proxy/main.py#L181-L186); [mem0/proxy/main.py:124](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/proxy/main.py#L124) (verified)
  - *To reach the next level:* No session scoping or human review before persistence.
- **Cap:** none

### C7 Third-party extensions — 0.12 (high)

The default configuration (OpenAI and local Qdrant) loads no third-party code at runtime. When the optional NLP extra is installed, mem0 downloads and installs the en_core_web_sm spaCy package by itself on first use, with no prompt. The optional HuggingFace embedder and reranker load whatever model name is configured, with no revision pinning or integrity check. All of these load inside the application's process, with its keys and environment.

- **S L1:** Model sources are developer-chosen but unpinned (no revision/hash); spaCy model version is resolved at download time. — [mem0/embeddings/huggingface.py:27](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/embeddings/huggingface.py#L27); [mem0/reranker/huggingface_reranker.py:58-59](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/reranker/huggingface_reranker.py#L58-L59); [mem0/utils/spacy_models.py:30-35](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/utils/spacy_models.py#L30-L35) (verified)
  - *To reach the next level:* No version pinning or integrity verification of downloaded models.
- **C L0:** No extension type (HF models, spaCy model) is verified. — [mem0/utils/spacy_models.py:30-35](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/utils/spacy_models.py#L30-L35); [mem0/embeddings/huggingface.py:27](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/embeddings/huggingface.py#L27) (verified)
  - *To reach the next level:* No verification on any type.
- **D L1:** Nothing third-party loads in the default config, but installing the [nlp] extra triggers an automatic, unprompted spaCy model install on first use. — [mem0/utils/spacy_models.py:30-35](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/utils/spacy_models.py#L30-L35); [mem0/llms/configs.py:7](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/llms/configs.py#L7); [mem0/vector_stores/configs.py:9](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/vector_stores/configs.py#L9) (verified)
  - *To reach the next level:* Model downloads happen without showing what will be installed.
- **B L0:** Downloaded models and the spaCy package load in-process with all the application's credentials and environment. — [mem0/utils/spacy_models.py:30-35](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/utils/spacy_models.py#L30-L35); [mem0/embeddings/huggingface.py:27](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/embeddings/huggingface.py#L27) (verified)
  - *To reach the next level:* No process separation or sandboxing for loaded models.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.25 (high)

API keys come from configuration or environment variables and are never put into prompts. A redaction helper blanks secret-looking fields when configs are cloned, and the server redacts config output. Telemetry to PostHog is on by default. It sends no memory content, but it does send memory IDs, unsalted MD5 hashes of user IDs, system details, and for the hosted client the account email. Raw conversation messages are stored in plaintext in ~/.mem0/history.db, and some warnings log whole message objects.

- **S L1:** Secrets come from env/config; one redaction helper masks sensitive config fields on the config-clone path only. — [mem0/llms/openai.py:50](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/llms/openai.py#L50); [mem0/memory/main.py:254-267](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L254-L267) (verified)
  - *To reach the next level:* No log or transcript redaction and no secret store or encryption at rest.
- **C L1:** Redaction covers config cloning; logs (full message objects in warnings), the plaintext messages/history DB and telemetry identifiers are unprotected. — [mem0/memory/main.py:890](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L890); [mem0/configs/base.py:44](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/configs/base.py#L44) (verified)
  - *To reach the next level:* Logs and stored transcripts are not redacted.
- **D L1:** Telemetry is on by default (MEM0_TELEMETRY defaults True), mostly content-free but sends memory IDs, unsalted md5(user_id) and, for the hosted client, the user email. — [mem0/memory/telemetry.py:14](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/telemetry.py#L14); [mem0/memory/utils.py:244](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/utils.py#L244); [mem0/memory/telemetry.py:96](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/telemetry.py#L96) (verified)
  - *To reach the next level:* Telemetry not opt-in.
- **B L1:** A leak exposes long-lived LLM provider and vector-store keys; the model has no tool to reach them and no subprocess inherits them. — [mem0/llms/openai.py:50](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/llms/openai.py#L50) (verified)
  - *To reach the next level:* Keys are long-lived and not scoped per task.
- **Cap:** none

### C9 Audit & traceability — 0.50 (high)

Every memory add, update and delete is written to a SQLite history table with old and new text, the event type, timestamps, and actor and role fields (not filled in on the batch-add path). It is on by default and stored in ~/.mem0, outside any workspace. The record does not note which user scope a change belongs to. reset() drops the whole table, the entity store has no history, and if writing a history row fails, mem0 logs an error and keeps the memory anyway.

- **S L2:** Structured per-mutation record (memory_id, old/new text, event, timestamps, actor_id, role). — [mem0/memory/storage.py:108-119](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/storage.py#L108-L119) (verified)
  - *To reach the next level:* No requesting-principal or scope attribution, and no correlation IDs.
- **C L2:** Add, update and delete of memories are recorded; entity-store link updates are not. — [mem0/memory/storage.py:108-119](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/storage.py#L108-L119); [mem0/memory/main.py:1080-1086](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L1080-L1086) (verified)
  - *To reach the next level:* Entity-store changes and config changes are not recorded.
- **D L2:** On by default at ~/.mem0/history.db, outside the workspace, but the same process can drop it via reset(). — [mem0/configs/base.py:44](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/configs/base.py#L44); [mem0/memory/storage.py:333](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/storage.py#L333) (verified)
  - *To reach the next level:* History is writable and deletable by the library process (reset drops the table).
- **B L2:** History write failures are logged at error level per record, but the memory insert has already happened and proceeds. — [mem0/memory/main.py:1093-1098](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L1093-L1098) (verified)
  - *To reach the next level:* Writes are not refused when their history record fails.
- **Cap:** none

### C10 Limits & kill switch — 0.50 (high)

Mem0 has no agent loop. Each add() makes one bounded extraction call (10 existing memories, the last 10 messages, max_tokens 2000 by default), and the model can't raise these limits. There is no wall-clock limit beyond the provider SDK's timeouts, no cost cap, and no cancel. The proxy starts a new background thread for every request with no limit on how many run at once.

- **S L2:** Fixed single LLM call per operation (no loop) plus a default max_tokens cap enforced in code. — [mem0/configs/llms/base.py:21](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/configs/llms/base.py#L21); [mem0/memory/main.py:922](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/memory/main.py#L922) (verified)
  - *To reach the next level:* No wall-clock or cost cap and no rate limits.
- **C L2:** Every model-call path (extraction, vision description, procedural memory) is single-shot and token-capped. — [mem0/proxy/main.py:161](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/proxy/main.py#L161); [mem0/configs/llms/base.py:21](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/configs/llms/base.py#L21) (verified)
  - *To reach the next level:* Proxy background threads are unbounded and not counted against any budget.
- **D L2:** Sensible token default, operator-configurable; the model cannot change it. — [mem0/configs/llms/base.py:21](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/configs/llms/base.py#L21) (verified)
  - *To reach the next level:* No hard ceiling configuration can't exceed.
- **B L2:** Runaway cost per call is bounded, but in-flight calls and proxy daemon threads run to completion with no cancel. — [mem0/proxy/main.py:161](https://github.com/mem0ai/mem0/blob/abb81c88e1f738a8117d8293530fbc31a5ef8fd9/mem0/proxy/main.py#L161) (verified)
  - *To reach the next level:* No cancellation of in-flight calls or provider-side budget.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Any conversation content passed to add() enters the extraction prompt (mem0/configs/prompts.py:1040) · [B] sensitive data/systems: The user's stored memories and last-10 messages of the same scope are in the same prompt (mem0/memory/main.py:922-940) · [C] state change / egress: Inferred memories are persisted automatically (mem0/memory/main.py:1057) and re-served to the host with its tools (mem0/proxy/main.py:124,181-186) · Same default session? Yes

## Highest-impact improvements
1. Check that the memory belongs to the caller's user_id/agent_id/run_id in get, update, delete and history. — C1 C L1→L2, +0.075 before caps (Playbook 4)
2. Make PostHog telemetry opt-in, and salt or drop user-ID hashes and emails. — C8 D L1→L2, +0.050 before caps
3. Return provenance (source role, actor, untrusted flag) with every memory, and have the proxy present memories as quoted data rather than user-turn text. — C6 S L1→L2, +0.075 before caps (Playbook 2)
4. Add an optional review or approval callback, or a pending state, for inferred memories before they are saved. — C2 S L0→L2, +0.150 before caps (Playbook 5)
5. Stop auto-installing the spaCy model; require an explicit install and pin HuggingFace model revisions. — C7 D L1→L2, +0.050 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scope is the core Python SDK (mem0/). The self-hosted FastAPI server (server/), TypeScript SDK (mem0-ts/), CLIs (cli/), agent/editor plugins and MCP-style integrations (integrations/), and the hosted platform were not scored. Server footnote: auth is on by default, but its authorization is not locked down, and server/main.py:43 calls load_dotenv() on the working directory.
- The vector-store providers (about 27) were sampled (qdrant default, pgvector); others may differ in filter handling.
- Third-party library behaviour inferred, not verified in their source: spacy.cli.download pip-installs the model wheel from spaCy's GitHub releases; openai SDK default timeouts and retries.
- Runtime notice copy is fetched from mem0's own GitHub main branch (mem0/memory/notices.py:22-23) and may appear in exception messages. It is text, not code, from the vendor's official source.
- No text aimed at AI reviewers was found. The README contains instructions aimed at AI agents ('Sign up as an agent', README.md:88-107); they were treated as data and not followed.
