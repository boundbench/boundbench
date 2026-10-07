# Defense-in-Depth Score: MCP for Blender

**Repo:** https://github.com/ahujasid/mcp-for-blender · **Commit:** `addda778d58a6a05f10a61ef8e2ca8958e915637` · **Reviewed:** 2026-10-05
**What it is:** MCP server controlling Blender, including arbitrary Python execution
**Category:** AI Assistants
**Scored configuration:** stdio server launched with `uvx mcp-for-blender` (package version 2.1.6) and the bundled Blender addon on localhost:9876, no environment variables set: safe mode off, asset and generation integrations off, anonymous usage telemetry on.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials opt-in · persistent memory no · untrusted input yes · third party extensions opt-in · sub agents no · external communication opt-in

## Score: 1.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C3 | Tool & action scoping | L0 | L1 | L0 | L0 | 0.07 | — | **0.07** | High |
| C4 | Code-execution isolation | L1 | L1 | L0 | L0 | 0.15 | G1 | **0.15** (alt) | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L0 | 0.07 | C5-WORSTCASE | **0.07** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C9 | Audit & traceability | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | High |
| C10 | Limits & kill switch | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |


The central tool runs any Python the model writes inside the user's Blender, with the user's full file, network and credential access and no sandbox. The server leaves approval entirely to the MCP host, marks only its read-only viewing tools, and gives untrusted asset or scene text no special handling, so one injected instruction can read and send private data or make irreversible changes. An opt-in safe mode filters scripts before they run, but it is off by default and is a filter, not a boundary. Logging, limits and secret handling are thin.

## Critical gaps
- Model-written Python runs with the operating-system user's full authority; nothing narrows identity or credentials. (ASI03, T3; C1) — [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248)
- Code execution is in-process exec() inside Blender with no isolation by default. (ASI05, T11; C4) — [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248); [src/blender_mcp/safe_mode.py:77](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/safe_mode.py#L77)
- A hijacked session can both exfiltrate data and take irreversible actions through the code tool, with no server-side human step (C5-WORSTCASE). (ASI01, LLM01, T6; C5) — [src/blender_mcp/server.py:771-773](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L771-L773); [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248)
- Integration switches and an API endpoint are read from properties of the opened .blend file without a trust decision (C6-REPOCONFIG). (ASI06, T1; C6) — [src/blender_mcp/bundled/addon.py:1200-1208](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L1200-L1208); [src/blender_mcp/bundled/addon.py:1525](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L1525)
- Appended third-party asset data loads into the Blender process, which holds full user authority. (ASI04, T17; C7) — [src/blender_mcp/bundled/addon.py:3142](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L3142)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

The server holds no identity of its own and never authenticates who is calling. Its main tool runs arbitrary Python inside the user's Blender process, which has the full authority of the logged-in operating-system user: every file in the home directory, the network, and the third-party API keys stored in the addon preferences. Nothing narrows that authority by default, and the Blender addon accepts commands from any local process on its loopback port. A hijacked session therefore acts as the user on the whole machine.

- **S L0:** execute_blender_code forwards model-written Python that the addon runs with exec() inside Blender, with the OS user's full ambient authority. — [src/blender_mcp/server.py:771-773](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L771-L773); [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248) (verified)
  - *To reach the next level:* L1 needs a dedicated or scoped identity; the code runs as the launching user with no narrowing.
- **C L0:** No authorization check exists on any path; the addon's command dispatcher maps execute_code straight to exec for any connected client. — [src/blender_mcp/bundled/addon.py:1509](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L1509); [src/blender_mcp/safe_mode.py:12-13](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/safe_mode.py#L12-L13) (verified)
  - *To reach the next level:* L1 needs at least the main tool path to pass an authorization check.
- **D L0:** Arbitrary code execution is the default tool surface; least privilege would need the operator to opt into safe mode, which is a validator rather than a privilege reduction. — [src/blender_mcp/safe_mode.py:4](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/safe_mode.py#L4); [src/blender_mcp/safe_mode.py:77](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/safe_mode.py#L77) (verified)
  - *To reach the next level:* L1 needs a narrower default identity; the default runs with the user's full privileges.
- **B L0:** Hijacked code reaches the user's entire account on the machine (home directory, SSH and cloud credential files, network) plus the stored Sketchfab, Hyper3D, Tencent and Premium keys. — [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248); [src/blender_mcp/bundled/addon.py:1200-1208](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L1200-L1208) (verified)
  - *To reach the next level:* L1 would need authority limited to fewer systems than the user's whole account.
- **Cap:** none

### C2 Approval gates — 0.15 (high)

As a tool server, the project leaves approval to the MCP host and gives the host little to work with. Four read-only tools carry a read-only hint, but the tools that change state, including the arbitrary-code tool, carry no risk annotation, so hosts fall back to the protocol's generic default. The code tool mixes reads and writes in one call, and there is no preview, dry run, read-only mode or server-side confirmation. Code that runs can delete files, overwrite saved work or call paid generation APIs, none of which Blender's undo reverses.

- **S L1:** Read-only hints exist on search_mentions, open_viewport, viewport_latest and viewport_pick, but mutating tools (execute_blender_code, import_asset, generate_3d) have no annotations and the main tool mixes reads and writes. — [src/blender_mcp/server.py:1945](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L1945); [src/blender_mcp/server.py:1968-1970](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L1968-L1970); [src/blender_mcp/server.py:771](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L771) (verified)
  - *To reach the next level:* L2 needs separate read and write tools with accurate hints on every tool.
- **C L1:** Risk signalling covers only the read-only viewing tools; the most powerful tool relies on the protocol default with no explicit destructive hint. — [src/blender_mcp/server.py:771-773](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L771-L773); [src/blender_mcp/server.py:1945](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L1945) (verified)
  - *To reach the next level:* L2 needs every built-in tool, including the code tool, to carry accurate risk signalling.
- **D L0:** The server offers no confirmation step or read-only mode; whether anything is approved depends entirely on the host. — [src/blender_mcp/server.py:771-773](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L771-L773); [src/blender_mcp/server.py:801](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L801) (verified)
  - *To reach the next level:* L1 needs a server-side gate or confirmation that is on by default.
- **B L0:** A wrongly approved script can delete or overwrite files, send data over the network or spend on paid generation with no undo outside Blender's scene history. — [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248) (verified)
  - *To reach the next level:* L1 needs most consequential actions to be reversible or bounded.
- **Cap:** none

### C3 Tool & action scoping — 0.07 (high)

The primary tool takes an arbitrary Python string and runs it, so there is no argument scoping on the path that matters most. The narrower tools do validate inputs: result limits and wait times are clamped, generation quality is an enum, and downloaded archives are checked for path traversal before extraction. The image argument to 3D generation accepts any local file path and uploads that file to the chosen provider. Every tool, including code execution, is enabled by default.

- **S L0:** execute_blender_code is raw passthrough of model-written Python to exec(). — [src/blender_mcp/server.py:771-773](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L771-L773); [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248) (verified)
  - *To reach the next level:* L1 needs at least a denylist or filter on the main tool's input in the default configuration.
- **C L1:** Secondary tools clamp limits and validate archive paths, but the code tool has no validation by default. — [src/blender_mcp/server.py:1819](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L1819); [src/blender_mcp/bundled/addon.py:4106](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L4106); [src/blender_mcp/generation.py:113-118](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/generation.py#L113-L118) (verified)
  - *To reach the next level:* L2 needs most tools, including the main one, to validate their inputs.
- **D L0:** The arbitrary-code tool is registered unconditionally and cannot be disabled by configuration. — [src/blender_mcp/server.py:771-773](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L771-L773) (verified)
  - *To reach the next level:* L1 needs dangerous tools to be individually disableable.
- **B L0:** A misused code tool can do anything the Blender process can do on the whole machine. — [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248) (verified)
  - *To reach the next level:* L1 needs reach narrower than the whole machine.
- **Cap:** none

### C4 Code-execution isolation — 0.15 (high)

Model-written Python runs in-process inside Blender through exec(), with no sandbox, separate user or container. An opt-in safe mode, turned on with an environment variable, checks scripts against an allowlist of syntax, modules and Blender paths before they are sent; its own documentation describes it as a guard on what the model can be talked into, not a sandbox, and it covers only the MCP path. Even with safe mode on, a script that escapes it lands in the same process with the user's full authority. The criterion takes the opt-in safe mode's score, capped because it is off by default.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Scripts run via in-process exec() in Blender's main interpreter with no isolation primitive. — [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248); [src/blender_mcp/safe_mode.py:4](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/safe_mode.py#L4) (verified)
    - *To reach the next level:* L1 needs at least filtering on the default path; nothing filters by default.
  - **C L0:** No execution path is isolated in the default configuration. — [src/blender_mcp/server.py:801](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L801); [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248) (verified)
    - *To reach the next level:* L1 needs the main exec tool to go through an isolation layer.
  - **D L0:** Isolation is absent by default; safe mode is opt-in through BLENDER_MCP_SAFE_MODE. — [src/blender_mcp/safe_mode.py:77](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/safe_mode.py#L77) (verified)
    - *To reach the next level:* L1 needs a containment layer that is on by default.
  - **B L0:** Code runs inside the Blender process with the user's home directory, network and stored API keys reachable. — [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248); [src/blender_mcp/bundled/addon.py:1200-1208](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L1200-L1208) (verified)
    - *To reach the next level:* L1 needs at least the home directory or credentials kept out of reach.
- **opt-in safe mode (BLENDER_MCP_SAFE_MODE)** (alt; raw 0.15, cap G1 → 0.15) ← counted
  - **S L1:** Safe mode is a deny-by-default AST walk with module, builtin, attribute and Blender-path allowlists, applied in-process before sending; it is a filter on a full-capability runtime. — [src/blender_mcp/safe_mode.py:1-12](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/safe_mode.py#L1-L12); [src/blender_mcp/safe_mode.py:472](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/safe_mode.py#L472) (verified)
    - *To reach the next level:* L2 needs OS-level separation (container, low-privilege user); AST restrictions remain escapable filtering.
  - **C L1:** The check runs only in the MCP server's execute_blender_code path; the addon socket accepts execute_code from any local process without it, and safe mode by design still permits file reads and writes through Blender's import, export, save and open operators. — [src/blender_mcp/server.py:783](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L783); [src/blender_mcp/safe_mode.py:12-13](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/safe_mode.py#L12-L13); [src/blender_mcp/safe_mode.py:25-26](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/safe_mode.py#L25-L26) (verified)
    - *To reach the next level:* L2 needs most execution paths covered with only low-power exceptions.
  - **D L0:** Off unless the operator sets BLENDER_MCP_SAFE_MODE. — [src/blender_mcp/safe_mode.py:77](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/safe_mode.py#L77) (verified)
    - *To reach the next level:* L1 needs the mechanism on by default.
  - **B L0:** A script that passes or escapes the validator runs in the same Blender process with full user authority. — [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248) (verified)
    - *To reach the next level:* L1 needs the process to lack home-directory or credential access.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.07 (high)

Untrusted text reaches the model from asset-library search results (third-party model names and author names), from object and material names in whatever .blend file is open, and from viewport images. The server returns this as plain text alongside its own guidance, with no provenance or untrusted marking, and offers no read-only or no-egress mode. Because the same session can run arbitrary Python, a successful injection can read private files and keys, send them anywhere and make irreversible changes, unless the host's approval stops it.

- **S L1:** Search results are formatted strings that interleave third-party names and authors with server-written text, with no provenance flag. — [src/blender_mcp/server.py:1112-1120](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L1112-L1120) (verified)
  - *To reach the next level:* L2 needs structured outputs separating returned content from metadata.
- **C L0:** No untrusted source is distinguished: asset results, scene data and images all enter context the same way. — [src/blender_mcp/server.py:1106](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L1106) (verified)
  - *To reach the next level:* L1 needs at least one untrusted source handled differently.
- **D L0:** No containment exists to turn on. — [src/blender_mcp/server.py:771-773](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L771-L773) (verified)
  - *To reach the next level:* L1 needs a containment mechanism enabled by default.
- **B L0:** A hijacked session can run arbitrary Python that both exfiltrates data and makes irreversible changes, with nothing in the server requiring a human. — [src/blender_mcp/server.py:771-773](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L771-L773); [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248) (verified)
  - *To reach the next level:* L1 needs at least one of exfiltration or irreversible action to be impossible without a human.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.10 (high)

The server keeps no memory store, but its code tool can write state that runs again later: Blender handlers, timers, registered scripts, the startup file and installed addons are all reachable when safe mode is off. Integration switches (asset libraries, 3D generators) and some endpoint and key settings are read from properties of the currently open scene, which Blender saves inside .blend files, so opening a file can turn integrations on without a separate trust prompt. Nothing records or reviews these persistent changes.

- **S L0:** Model-written code can register persistent handlers, timers and scripts or overwrite the startup file; safe mode, which blocks these, is off by default. — [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248); [src/blender_mcp/safe_mode.py:38](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/safe_mode.py#L38) (verified)
  - *To reach the next level:* L1 needs persistent writes to be at least logged.
- **C L0:** Neither model-written persistence nor scene-file-supplied configuration is controlled. — [src/blender_mcp/bundled/addon.py:1200-1208](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L1200-L1208); [src/blender_mcp/bundled/addon.py:1525](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L1525) (verified)
  - *To reach the next level:* L1 needs at least one persistence path controlled.
- **D L1:** Single-user local design with no namespace concept; scene-level settings override environment configuration for whoever opens the file. — [src/blender_mcp/bundled/addon.py:1201](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L1201) (verified)
  - *To reach the next level:* L2 needs per-session isolation enforced in code.
- **B L1:** Persistent code written by the model runs in every later Blender session of the same user. — [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248) (verified)
  - *To reach the next level:* L2 needs persistence that can only influence text output or gated actions.
- **Cap:** C6-REPOCONFIG — Integration toggles and the Hunyuan3D API URL are read from scene properties of the open .blend file, so a file the user opens can enable integrations and redirect that endpoint without a trust decision (Blender saving scene properties in .blend files is the library behaviour relied on).

### C7 Third-party extensions — 0.15 (high)

The server has no plugin system. The third-party content it loads at runtime is downloaded assets: Poly Haven models are appended from .blend files, checked against the md5 the Poly Haven API publishes, while Sketchfab, Poly Pizza and generator results are mesh formats. Integrations are off by default but can be switched on by the settings stored in an opened scene, and appended .blend data runs inside the Blender process; whether embedded scripts in it run depends on Blender's own auto-run setting. Separately, the addon checks the project's main branch for updates and installs them on a user click without signature verification.

- **S L1:** Poly Haven .blend downloads come from the official API over HTTPS with an md5 the same API supplies; nothing is pinned to a version or signed. — [src/blender_mcp/bundled/addon.py:473](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L473); [src/blender_mcp/bundled/addon.py:3142](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L3142) (verified)
  - *To reach the next level:* L2 needs pinned versions of loaded assets.
- **C L1:** Only the Poly Haven path carries any integrity check. — [src/blender_mcp/bundled/addon.py:473](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L473) (verified)
  - *To reach the next level:* L2 needs most loaded content types covered.
- **D L0:** Integration toggles live in scene properties, so an opened .blend file can enable asset sources without a separate prompt. — [src/blender_mcp/bundled/addon.py:1525](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L1525); [src/blender_mcp/bundled/addon.py:6235-6238](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L6235-L6238) (verified)
  - *To reach the next level:* L1 needs enabling an integration to require explicit user consent that a workspace file cannot supply.
- **B L0:** Appended asset data is loaded into the Blender process itself, which holds the user's full authority. — [src/blender_mcp/bundled/addon.py:3142](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L3142) (verified)
  - *To reach the next level:* L1 needs loaded content handled in a separate process.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.20 (high)

Third-party keys for Sketchfab, Hyper3D, Poly Pizza, Tencent Hunyuan3D and the Premium licence are read from addon preferences, scene properties or environment variables; preferences and scene properties are stored in Blender's own files, and scene-level keys travel inside saved .blend files. Most key fields are masked in the UI, but nothing keeps them away from the model, which can read them with one line of Python. The server logs every command with its full parameters at INFO level. Anonymous usage telemetry (tool name, success, duration) is on by default, while prompts, code and screenshots are sent only after an explicit opt-in.

- **S L1:** Keys come from preferences, scene properties or environment variables; masking is the UI password subtype only, and logs record full command parameters. — [src/blender_mcp/bundled/addon.py:1200-1208](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L1200-L1208); [src/blender_mcp/bundled/addon.py:6288-6291](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L6288-L6291); [src/blender_mcp/server.py:226](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L226) (verified)
  - *To reach the next level:* L2 needs log filters or masked secret types on main paths.
- **C L1:** Only the telemetry path strips content and error details when consent is absent; logs and model-reachable state are unprotected. — [src/blender_mcp/telemetry.py:214-221](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/telemetry.py#L214-L221) (verified)
  - *To reach the next level:* L2 needs logs and stored transcripts protected as well.
- **D L1:** Content-free usage telemetry is on by default (env vars disable it), content collection is opt-in, and INFO-level parameter logging is on by default. — [src/blender_mcp/telemetry.py:107-110](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/telemetry.py#L107-L110); [src/blender_mcp/bundled/addon.py:5723-5726](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L5723-L5726); [src/blender_mcp/server.py:52](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L52) (verified)
  - *To reach the next level:* L2 needs telemetry opt-in and default logging without full payloads.
- **B L0:** Long-lived third-party API keys and licence keys are readable by any code the model runs in Blender. — [src/blender_mcp/bundled/addon.py:1200-1208](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L1200-L1208); [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248) (verified)
  - *To reach the next level:* L1 needs keys kept out of reach of model-run code or scoped narrowly.
- **Cap:** none

### C9 Audit & traceability — 0.33 (high)

The only default record is the server's standard-error log, which prints each command sent to Blender with its full parameters (including the code) and the response status, with timestamps. It is unstructured, written wherever the MCP host captures server logs, and reachable by the code the model runs. The richer trajectory recording is opt-in and uploads to the project's hosted database rather than keeping a local audit trail. Commands sent to the addon socket by other local processes are not logged by the server.

- **S L1:** Unstructured INFO log lines record every command type, parameters and response status. — [src/blender_mcp/server.py:226](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L226); [src/blender_mcp/server.py:52](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L52) (verified)
  - *To reach the next level:* L2 needs a structured per-call record with arguments, status and timestamps.
- **C L2:** Every tool reaches Blender through send_command, which logs each call; direct socket clients bypass the server log. — [src/blender_mcp/server.py:226](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L226); [src/blender_mcp/bundled/addon.py:1450](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L1450) (verified)
  - *To reach the next level:* L3 needs all paths, including direct addon socket commands and approvals, recorded.
- **D L1:** Logging is on by default but lands where code running in Blender, with the user's file access, can alter it; trajectory capture is consent-gated and off-host. — [src/blender_mcp/server.py:226](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L226); [src/blender_mcp/trajectory.py:582](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/trajectory.py#L582) (verified)
  - *To reach the next level:* L2 needs the record stored where the agent's own tools cannot edit it.
- **B L1:** Logging is best-effort stream output with no durability guarantees. — [src/blender_mcp/server.py:226](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L226) (verified)
  - *To reach the next level:* L2 needs per-action flushed records with surfaced errors.
- **Cap:** none

### C10 Limits & kill switch — 0.38 (high)

The server waits at most 180 seconds for any Blender command and clamps several arguments, such as search limits and generation wait times. These are limits on waiting, not on work: the addon runs scripts on Blender's main thread with no timeout, so a long or looping script keeps running after the server gives up, and there is no cancel. There is no rate limit or spend cap on paid 3D generation calls.

- **S L2:** Server-enforced caps exist on some operations: a 180-second socket timeout, clamped result limits and generation wait times, and code-size limits in safe mode. — [src/blender_mcp/server.py:233](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L233); [src/blender_mcp/server.py:1721](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L1721); [src/blender_mcp/server.py:1819](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L1819) (verified)
  - *To reach the next level:* L3 needs caps on every operation plus rate or concurrency limits.
- **C L1:** The timeout only stops the server waiting; the in-process exec in Blender continues unbounded. — [src/blender_mcp/server.py:233](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L233); [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248) (verified)
  - *To reach the next level:* L2 needs tool timeouts that actually stop the work.
- **D L2:** Limits are hard-coded with sensible values, and model-supplied values are clamped. — [src/blender_mcp/server.py:1721](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/server.py#L1721) (verified)
  - *To reach the next level:* L3 needs limits the model cannot reset, including on work that outlives the timeout.
- **B L1:** A runaway script keeps running in Blender after the server times out, and generation spend has no ceiling. — [src/blender_mcp/bundled/addon.py:2243-2248](https://github.com/ahujasid/mcp-for-blender/blob/addda778d58a6a05f10a61ef8e2ca8958e915637/src/blender_mcp/bundled/addon.py#L2243-L2248) (verified)
  - *To reach the next level:* L2 needs stopping to end in-flight work and moderate spend ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Third-party asset names and authors in search results (src/blender_mcp/server.py:1112) and object names from any opened .blend file · [B] sensitive data/systems: User files and stored third-party API keys reachable from Blender (src/blender_mcp/bundled/addon.py:1200) · [C] state change / egress: Arbitrary Python via execute_blender_code (src/blender_mcp/bundled/addon.py:2248) · Same default session? Yes

## Highest-impact improvements
1. Annotate every tool accurately (destructiveHint on execute_blender_code, import_asset, generate_3d) and split read-only scene inspection from code execution. — C2 S L1→L2, +0.075 before caps (Playbook 5)
2. Turn safe mode on by default with an explicit, loudly named opt-out, and apply the same check in the addon so direct socket commands are covered. — C4 D L0→L2, +0.100 before caps (Playbook 3 step 1)
3. Read integration toggles and endpoint URLs only from addon preferences or the environment, never from scene properties stored in .blend files. — C6 C L0→L1, +0.075 before caps (Playbook 2)
4. Write a structured local JSON-lines record of every tool call (arguments, status, timestamp) outside directories the Blender process can easily write. — C9 S L1→L2, +0.075 before caps (Playbook 1 step 3)
5. Add a per-script execution timeout and cancellation in the addon and a per-session cap on paid generation calls. — C10 C L1→L2, +0.075 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The project was renamed: the queue listed https://github.com/ahujasid/blender-mcp, which resolves to the same HEAD as https://github.com/ahujasid/mcp-for-blender; GitHub repository search returns ahujasid/mcp-for-blender as the repository's current name and the README calls the project formerly blender-mcp, so the new URL is used. pyproject.toml and the addon still reference the old URL.
- src/blender_mcp/config.py (telemetry endpoint and settings) is gitignored and not in the repository, so telemetry configuration values were not examined.
- Behaviour of Blender itself (scene properties saved in .blend files, auto-run of embedded scripts, operator semantics) is inferred from Blender's documented behaviour, not from code in this repository.
- The hosted Premium generation service and the Supabase backend are outside the repository and were not examined.
- No reviewer-directed instructions were found in the repository text.
