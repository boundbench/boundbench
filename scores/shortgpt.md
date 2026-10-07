# Defense-in-Depth Score: ShortGPT

**Repo:** https://github.com/rayventura/shortgpt · **Commit:** `3df4e0f7a422bf7386565d498bf4521a2544c614` (v0.3.0) · **Reviewed:** 2026-10-04
**What it is:** AI video automation app that scripts, voices, captions and renders short-form and long-form videos (and dubs YouTube videos) behind a Gradio web UI.
**Category:** AI Assistants
**Scored configuration:** Local Gradio web app started by runShortGPT.py (or the README's Docker image with --env-file .env), default settings, keys supplied via .env or the Config tab.
**Agent surface (default):** code execution no · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents no · external communication no

## Score: 3.4 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L0 | L0 | L1 | 0.12 | — | **0.12** | High |
| C2 | Approval gates | L1 | L2 | L2 | L2 | 0.42 | — | **0.42** | High |
| C3 | Tool & action scoping | L2 | L1 | L3 | L2 | 0.47 | — | **0.47** | High |
| C4 | Code-execution isolation | L2 | L2 | L2 | L0 | 0.40 | — | **0.40** | High |
| C5 | Untrusted input blast radius | L2 | L2 | L3 | L3 | 0.60 | — | **0.60** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L3 | 0.40 | — | **0.40** | High |
| C7 | Third-party extensions | L2 | L2 | L0 | L0 | 0.30 | — | **0.30** | Medium |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | Medium |
| C9 | Audit & traceability | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |


ShortGPT is a fixed content pipeline, not a tool-using agent: the LLM only returns text (scripts, search queries, titles, translations), so a hijacked model cannot pick actions, run code or read secrets. The dominant risk is the web app around it: access control on its default exposure of the UI and stored API keys is not locked down. There are no spend or wall-clock limits, and two LLM retry loops have no upper bound.

## Critical gaps
- The documented Docker run passes every API key into the container environment with unrestricted network, so anything that executes there holds all credentials. (ASI05, T11; C4) — [README.md:82](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/README.md#L82); [shortGPT/editing_utils/handle_videos.py:61-63](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/editing_utils/handle_videos.py#L61-L63)
- Whisper model weights are auto-downloaded and deserialized in-process with the app's keys and environment, without consent. (ASI04, T17; C7) — [shortGPT/audio/audio_utils.py:73-76](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/audio/audio_utils.py#L73-L76)

## Criterion details

### C1 Identity & least privilege — 0.12 (high)

ShortGPT uses the operator's own OpenAI, Gemini, ElevenLabs and Pexels keys, each looked up directly by whichever module needs it. Access control on the web UI is not locked down, which exposes the operator's full key authority. Spawned ffmpeg processes inherit the whole environment. A stolen key gives full billing and account access on several paid services.

- **S L1:** Authority is the operator's dedicated but full-scope vendor API keys, used for every call. — [shortGPT/config/api_db.py:22-28](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/config/api_db.py#L22-L28); [shortGPT/gpt/gpt_utils.py:72-82](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_utils.py#L72-L82) (verified)
  - *To reach the next level:* No per-capability or read-only scoping of keys; each service gets one full key.
- **C L0:** There is no authorization layer: modules fetch keys themselves, and UI access control is not locked down. — searched `rg -n 'env='` in `shortGPT gui` → 0 hits (no subprocess call passes a scrubbed env; children inherit os.environ) (verified)
  - *To reach the next level:* No per-request authorization check of who is using the agent; subprocesses inherit the full environment.
- **D L0:** Access control on the default launch's network exposure is not locked down, and the README's Docker command puts all keys in the container environment. — [README.md:82](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/README.md#L82) (verified)
  - *To reach the next level:* No safe default for UI access.
- **B L1:** Hijacked or stolen credentials give write/spend access across several paid services (OpenAI or Gemini, ElevenLabs, Pexels). — [shortGPT/config/api_db.py:22-28](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/config/api_db.py#L22-L28) (verified)
  - *To reach the next level:* Keys are long-lived and span multiple services; nothing scopes them to one project or limits spend.
- **Cap:** none

### C2 Approval gates — 0.42 (high)

Every job starts with a human clicking a button in the UI with the inputs visible, and the model can never choose an action of its own, since it has no tools. But that one click unlocks the whole fixed pipeline, including paid API calls, with no step-level confirmation, and UI access control is not locked down, which affects who that human can be. Outputs are local files that are easy to undo, but API spend cannot be undone.

- **S L1:** Approval is one blanket human trigger per job (the Generate/Translate button) that runs all pipeline steps. — [shortGPT/engine/abstract_content_engine.py:64-74](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/engine/abstract_content_engine.py#L64-L74); [gui/ui_tab_short_automation.py:29](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/gui/ui_tab_short_automation.py#L29) (verified)
  - *To reach the next level:* No per-call approval showing the exact paid API call or file operation.
- **C L2:** No path lets the model start a consequential action; every API call, file write and render runs inside a human-started job. — searched `rg -n -i 'tools=|function_call|tool_choice'` in `shortGPT gui` → 0 hits (the LLM is never given tools; completions are plain chat calls); [shortGPT/engine/abstract_content_engine.py:64-74](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/engine/abstract_content_engine.py#L64-L74) (verified)
  - *To reach the next level:* Auto-run steps are not a verified read-only allowlist; paid calls run without their own gate.
- **D L2:** The trigger requirement is built into the UI and the model cannot turn it off, but UI access control is not locked down. (verified)
  - *To reach the next level:* Trigger is not tied to an authenticated principal.
- **B L2:** Common outcomes (files in videos/ and .editing_assets/) are reversible; API credit spend and asset deletion via unlink are not. — [shortGPT/config/asset_db.py:167](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/config/asset_db.py#L167); [gui/ui_tab_short_automation.py:29](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/gui/ui_tab_short_automation.py#L29); [shortGPT/engine/abstract_content_engine.py:49](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/engine/abstract_content_engine.py#L49) (verified)
  - *To reach the next level:* No spend or quantity bound on a wrongly started job (numShorts has no maximum).
- **Cap:** none
- **Notes:** UI access-control gaps are scored once, in C1, rather than as C2-SELFAPPROVE.

### C3 Tool & action scoping — 0.47 (high)

The model is given no tools; its outputs are used only as text, image-search query strings and timestamps. Timestamps and time ranges are range-checked, but search queries are pasted unencoded into the Bing URL, image URLs scraped from Bing are fetched with no host allowlist, and outbound fetching is not otherwise confined. The YouTube link check is a string-prefix match.

- **S L2:** Model outputs pass through typed parsing with some bounds (timestamps within audio length), but URLs are prefix-checked or not checked at all. — [shortGPT/gpt/gpt_editing.py:29](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_editing.py#L29); [gui/ui_tab_video_translation.py:108](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/gui/ui_tab_video_translation.py#L108); [shortGPT/api_utils/image_api.py:9](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/api_utils/image_api.py#L9); [shortGPT/editing_framework/core_editing_engine.py:203](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/editing_framework/core_editing_engine.py#L203) (verified)
  - *To reach the next level:* No host allowlist or internal-address blocking for fetched image URLs; YouTube URL uses startswith.
- **C L1:** Only the timestamp/time-range outputs are validated; query strings, Bing-returned URLs and Pexels links are used as-is. — [shortGPT/gpt/gpt_editing.py:29](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_editing.py#L29) (verified)
  - *To reach the next level:* Most model-derived and fetched values are not validated.
- **D L3:** The model receives no tools at all; every capability is fixed by pipeline code, and the model cannot enable more. — searched `rg -n -i 'tools=|function_call|tool_choice'` in `shortGPT gui` → 0 hits (the LLM is never given tools; completions are plain chat calls); [shortGPT/gpt/gpt_utils.py:92-97](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_utils.py#L92-L97) (verified)
  - *To reach the next level:* No per-task narrowing of the pipeline's own network and write capabilities.
- **B L2:** Misuse is limited to fetching attacker-influenced image URLs and writing files inside the project's working folders. — [shortGPT/editing_framework/core_editing_engine.py:203](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/editing_framework/core_editing_engine.py#L203); [shortGPT/engine/abstract_content_engine.py:49](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/engine/abstract_content_engine.py#L49) (verified)
  - *To reach the next level:* No quantity bounds; the LLM's list of image queries is not capped to the requested count.
- **Cap:** none

### C4 Code-execution isolation — 0.40 (high)

No model-generated code is ever executed; the app runs ffmpeg and ffprobe with fixed argument lists. Two helpers build shell command strings from file paths or URLs (ffprobe for caption aspect ratio, and an unused spleeter helper), a latent injection risk that Gradio's upload filename cleaning probably blunts. The documented Docker image is a stock root container with no hardening, started with every API key in its environment and full network, and the pip install path runs directly on the host.

- **S L2:** The README's run mode is a stock python:3.10-slim container running as root with no hardening; the pip/setup.py path has no isolation. — [Dockerfile:2-27](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/Dockerfile#L2-L27); [README.md:82](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/README.md#L82) (verified)
  - *To reach the next level:* No non-root user, dropped capabilities, seccomp or read-only root filesystem.
- **C L2:** In the Docker mode all exec paths (ffmpeg subprocesses, the shell=True ffprobe call) run inside the container. — [shortGPT/editing_utils/handle_videos.py:61-63](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/editing_utils/handle_videos.py#L61-L63); [shortGPT/audio/audio_utils.py:95-97](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/audio/audio_utils.py#L95-L97); [shortGPT/audio/audio_utils.py:49-53](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/audio/audio_utils.py#L49-L53) (verified)
  - *To reach the next level:* The pip-installed path runs everything on the host; shell strings are built from paths and URLs.
- **D L2:** Docker is the documented way to run locally, but nothing enforces it. — [README.md:82](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/README.md#L82) (verified)
  - *To reach the next level:* Container use is optional and undetected; nothing fails closed without it.
- **B L0:** Inside the container every API key is in the environment and network egress is unrestricted. — [README.md:82](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/README.md#L82); searched `rg -n 'env='` in `shortGPT gui` → 0 hits (no subprocess call passes a scrubbed env; children inherit os.environ) (verified)
  - *To reach the next level:* Credentials are passed into the execution environment.
- **Cap:** none
- **Notes:** The shell=True ffprobe call receives either a Gradio upload path (Gradio strips most special characters from upload filenames, inferred) or a googlevideo signed URL, so injection is latent rather than demonstrated; the spleeter helper is imported but never called.

### C5 Untrusted input blast radius — 0.60 (high)

Untrusted content does reach the model: YouTube audio transcripts are pasted into translation prompts, and image search results come from Bing. The structural limit is that the model has no tools and never sees API keys, so a hijacked completion can only change the text that gets spoken, captioned or searched. Nothing marks or isolates untrusted text, and UI access control is not locked down, but the worst case is altered video content and a search query, not secret theft or destructive actions.

- **S L2:** By design the LLM has no tools and no secrets in context, so injected text can only alter generated text and search queries; no explicit taint tracking. — [shortGPT/gpt/gpt_translate.py:8](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_translate.py#L8); searched `rg -n -i 'tools=|function_call|tool_choice'` in `shortGPT gui` → 0 hits (the LLM is never given tools; completions are plain chat calls); [shortGPT/gpt/gpt_utils.py:92-97](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_utils.py#L92-L97) (verified)
  - *To reach the next level:* No deliberate Rule-of-Two enforcement or quarantine of untrusted transcript text.
- **C L2:** The no-tools design applies to every LLM call (all go through llm_completion with plain system/user messages). — [shortGPT/gpt/gpt_utils.py:92-97](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_utils.py#L92-L97) (verified)
  - *To reach the next level:* Third parties are not distinguished from the principal.
- **D L3:** The constraint is architectural and nothing the model reads can configure tools or egress into existence. — searched `rg -n -i 'tools=|function_call|tool_choice'` in `shortGPT gui` → 0 hits (the LLM is never given tools; completions are plain chat calls) (verified)
  - *To reach the next level:* Not a declared, tested control; future pipeline changes could add tool use silently.
- **B L3:** A hijacked completion reaches only low-sensitivity text (user prompt, transcript) and its only outbound effect is a search query string or TTS text sent to fixed vendors. — [shortGPT/api_utils/image_api.py:57](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/api_utils/image_api.py#L57); [shortGPT/gpt/gpt_translate.py:8](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_translate.py#L8) (verified)
  - *To reach the next level:* A model-chosen query string still reaches Bing, an outbound channel.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.40 (high)

Job state (scripts, captions, file paths) is saved to a local TinyDB file per job and read back only when that job resumes; nothing the model writes is fed into other jobs. Keys and settings load from a .env file in the working directory, which here is the project's own install folder rather than a workspace someone else controls. The writes are not validated, but they only affect that job's output and the JSON files are easy to inspect and delete.

- **S L1:** Model output is persisted to the job document without validation and reloaded on resume. — [shortGPT/engine/abstract_content_engine.py:39-44](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/engine/abstract_content_engine.py#L39-L44); [shortGPT/database/db_document.py:42](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/database/db_document.py#L42) (verified)
  - *To reach the next level:* No provenance or validation on persisted job fields.
- **C L1:** No store is controlled; content_db, asset_db and api_db are plain TinyDB JSON files. — [shortGPT/database/db_document.py:42](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/database/db_document.py#L42); [shortGPT/config/api_db.py:5](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/config/api_db.py#L5) (verified)
  - *To reach the next level:* No control on any persistent store or on .env loading.
- **D L2:** Each job gets its own document id and asset directory, enforced in code. — [shortGPT/engine/abstract_content_engine.py:49](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/engine/abstract_content_engine.py#L49); [shortGPT/engine/abstract_content_engine.py:39-44](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/engine/abstract_content_engine.py#L39-L44) (verified)
  - *To reach the next level:* Model output is not prevented from changing paths stored in its own job document.
- **B L3:** Poisoned state is job-scoped, only influences that job's text and render, and sits in inspectable JSON under .database/. — [shortGPT/database/db_document.py:42](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/database/db_document.py#L42); [shortGPT/engine/abstract_content_engine.py:64-74](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/engine/abstract_content_engine.py#L64-L74) (verified)
  - *To reach the next level:* Nothing prompts a human to review persisted state before reuse.
- **Cap:** none

### C7 Third-party extensions — 0.30 (medium)

ShortGPT has no plugin or MCP system. The one piece of third-party code it loads at runtime is the Whisper speech model, downloaded automatically on first use and loaded inside the app process. The model name is fixed in code and the whisper library fetches it from OpenAI's own servers with a checksum (inferred), so the source is reasonable, but there is no consent step and the weights run with everything the app holds.

- **S L2:** A fixed Whisper model name is loaded through whisper_timestamped, which downloads from OpenAI's official URL with a SHA-256 check (inferred from the library). — [shortGPT/audio/audio_utils.py:73-76](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/audio/audio_utils.py#L73-L76); [requirements.txt:13-15](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/requirements.txt#L13-L15) (inferred)
  - *To reach the next level:* Verification lives in the dependency, not this code; unpinned whisper/torch versions in requirements.
- **C L2:** Whisper weights are the only runtime-loaded third-party artifact, and they go through the library's verified download. — [shortGPT/audio/audio_utils.py:73-76](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/audio/audio_utils.py#L73-L76) (inferred)
  - *To reach the next level:* Verification is not asserted in this repository's code.
- **D L0:** Weights are downloaded and loaded automatically on first transcription with no consent or display. — [shortGPT/audio/audio_utils.py:73-76](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/audio/audio_utils.py#L73-L76) (verified)
  - *To reach the next level:* No explicit install or display of what will be loaded.
- **B L0:** The model is deserialized in-process with the app's keys and full environment. — [shortGPT/audio/audio_utils.py:73-76](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/audio/audio_utils.py#L73-L76); [shortGPT/config/api_db.py:22-28](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/config/api_db.py#L22-L28) (verified)
  - *To reach the next level:* No separate, scrubbed process for model loading.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.20 (medium)

API keys come from a .env file or a plaintext TinyDB file, and are never placed in model prompts. But the UI's handling of stored keys is not locked down, and child processes inherit the full environment. LLM prompt/response logs contain no keys, and error stack traces are rendered into the UI. Nothing is redacted anywhere.

- **S L1:** Secrets come from env vars or a plaintext JSON store; there is no real masking. — [shortGPT/config/api_db.py:22-28](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/config/api_db.py#L22-L28) (verified)
  - *To reach the next level:* No redaction layer or secret store.
- **C L1:** Keys stay out of model-bound messages by construction; the subprocess environment and error views are unprotected, and UI key handling is not locked down. — [shortGPT/gpt/gpt_utils.py:92-97](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_utils.py#L92-L97); searched `rg -n 'env='` in `shortGPT gui` → 0 hits (no subprocess call passes a scrubbed env; children inherit os.environ) (verified)
  - *To reach the next level:* Logs, UI and subprocess env are not covered.
- **D L1:** No telemetry is configured in this code, but Gradio analytics are left at the library default (inferred on, content-free) and verbose LLM logs are always on. — searched `rg -n -i 'analytics_enabled|GRADIO_ANALYTICS'` in `gui shortGPT` → 0 hits (Gradio analytics left at the library default); [shortGPT/gpt/gpt_utils.py:108-112](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_utils.py#L108-L112) (inferred)
  - *To reach the next level:* Telemetry not explicitly opt-in; prompt/response logs always written.
- **B L0:** Long-lived full-account vendor keys are readable by every subprocess, and UI key handling is not locked down. — searched `rg -n 'env='` in `shortGPT gui` → 0 hits (no subprocess call passes a scrubbed env; children inherit os.environ); [shortGPT/config/api_db.py:5](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/config/api_db.py#L5) (verified)
  - *To reach the next level:* Keys are not scoped or short-lived and are exposed broadly.
- **Cap:** none

### C9 Audit & traceability — 0.30 (high)

Every LLM call writes a text file with the system prompt, user prompt and response under .logs/gpt_logs, and job progress lives in the TinyDB job document. Subprocess runs, network fetches, file deletions and who started a job are not recorded. The model cannot touch the logs since it has no tools, but they are plain files the app can overwrite.

- **S L1:** Unstructured per-call text files for LLM completions only. — [shortGPT/gpt/gpt_utils.py:108-112](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_utils.py#L108-L112) (verified)
  - *To reach the next level:* No structured record of every action with arguments, status and actor.
- **C L1:** Only the LLM call path is recorded. — [shortGPT/gpt/gpt_utils.py:108-112](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_utils.py#L108-L112); [shortGPT/config/asset_db.py:167](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/config/asset_db.py#L167) (verified)
  - *To reach the next level:* Subprocesses, fetches, file moves/deletes and key changes are not logged.
- **D L2:** Logging is on by default and the model has no way to edit it, but the app process can. — [shortGPT/gpt/gpt_utils.py:108-112](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_utils.py#L108-L112); searched `rg -n -i 'tools=|function_call|tool_choice'` in `shortGPT gui` → 0 hits (the LLM is never given tools; completions are plain chat calls) (verified)
  - *To reach the next level:* Not written by a component separate from the app.
- **B L1:** Logs are written per call but are best-effort and miss most actions. — [shortGPT/gpt/gpt_utils.py:108-112](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_utils.py#L108-L112) (verified)
  - *To reach the next level:* No durable per-action record that allows replay.
- **Cap:** none

### C10 Limits & kill switch — 0.20 (high)

Each LLM request has a 30-second timeout, a 2000-token cap and five retries, and the pipeline has a fixed list of steps. But two script-generation loops retry the LLM forever if the output never parses or stays too long, the number of shorts per click has no upper bound, ffmpeg runs have no timeouts, and there is no overall time or spend limit. Stopping means killing the server process.

- **S L1:** Per-request timeout and retry counts exist, but loops around them are unbounded and there is no job time or cost cap. — [shortGPT/gpt/gpt_utils.py:86-103](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_utils.py#L86-L103); [shortGPT/gpt/gpt_chat_video.py:7](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_chat_video.py#L7); [shortGPT/engine/reddit_short_engine.py:26](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/engine/reddit_short_engine.py#L26) (verified)
  - *To reach the next level:* No enforced iteration cap on the retry loops and no wall-clock or spend cap.
- **C L1:** Limits apply only to individual LLM requests. — [shortGPT/gpt/gpt_utils.py:86-103](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_utils.py#L86-L103); [shortGPT/audio/audio_utils.py:49-53](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/audio/audio_utils.py#L49-L53) (verified)
  - *To reach the next level:* Subprocesses (ffmpeg) and outer loops are not covered.
- **D L1:** Defaults exist per request but the user-controlled shorts count is unbounded. — [gui/ui_tab_short_automation.py:29](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/gui/ui_tab_short_automation.py#L29) (verified)
  - *To reach the next level:* No sensible job-level defaults or ceilings.
- **B L0:** A runaway generateScript loop can call the paid LLM indefinitely; stop is only a SIGINT that closes the server. — [shortGPT/gpt/gpt_chat_video.py:7](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/shortGPT/gpt/gpt_chat_video.py#L7); [gui/gui_gradio.py:48-54](https://github.com/rayventura/shortgpt/blob/3df4e0f7a422bf7386565d498bf4521a2544c614/gui/gui_gradio.py#L48-L54) (verified)
  - *To reach the next level:* No ceiling on spend or runtime.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Bing image-search HTML and result URLs (shortGPT/api_utils/image_api.py:57), YouTube audio transcripts fed to the LLM (shortGPT/gpt/gpt_translate.py:8) · [B] sensitive data/systems: OpenAI/Gemini/ElevenLabs/Pexels API keys present in the process environment (shortGPT/config/api_db.py:5) · [C] state change / egress: Outbound fetches of Bing-chosen image URLs (shortGPT/editing_framework/core_editing_engine.py:203), file writes under videos/ and .editing_assets/, paid API calls; the LLM itself has no tools · Same default session? Yes

## Highest-impact improvements
1. Harden the web UI's default network exposure and access control. — C1 D L0→L2, +0.100 before caps (Playbook 4)
2. Harden how stored API keys are handled in the UI. — C8 B L0→L1, +0.050 before caps (Playbook 4)
3. Cap the generateScript/correctScript and realisticness loops, add a max on numShorts, and add a per-job wall-clock and token budget. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
4. Run ffmpeg/ffprobe with argv lists only (drop shell=True) and pass a minimal env= to subprocesses. — C4 B L0→L1, +0.050 before caps (Playbook 3)
5. Add a non-root USER and drop capabilities in the Dockerfile; remove RUN printenv. — C4 S L2→L3, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The Colab notebook the README recommends is hosted outside the repository and was not examined; in-repo Colab mode (runShortGPTColab.py) was noted as more exposed than the scored local mode.
- Behaviour of third-party libraries (Gradio upload filename sanitisation, Gradio default analytics, whisper checkpoint SHA-256 verification, torch.load defaults, moviepy/imageio URL fetching) was inferred from knowledge of those libraries, not read in this repository.
- No text aimed at AI reviewers was found in the repository.
