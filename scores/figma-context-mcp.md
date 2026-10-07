# Defense-in-Depth Score: Framelink Figma MCP

**Repo:** https://github.com/GLips/Figma-Context-MCP · **Commit:** `c083d65c7e002923e7cb98f4e3bdafb105e90f6d` · **Reviewed:** 2026-10-05
**What it is:** MCP server (npm figma-developer-mcp) that fetches Figma file and node data, simplifies it for coding agents, and downloads image assets into a local directory.
**Category:** Coding
**Scored configuration:** Local stdio server launched as in the README (npx -y figma-developer-mcp --figma-api-key=KEY --stdio), with both tools registered, telemetry at its default and no --image-dir.
**Agent surface (default):** code execution no · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 4.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L1 | L3 | 0.50 | — | **0.50** | High |
| C2 | Approval gates | L1 | L1 | L1 | L2 | 0.30 | — | **0.30** | High |
| C3 | Tool & action scoping | L2 | L3 | L1 | L2 | 0.53 | — | **0.53** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L2 | 0.17 | — | **0.17** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | SA | L2 | 0.30 | C6-REPOCONFIG | **0.25** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L2 | L3 | L1 | L1 | 0.47 | — | **0.47** | High |
| C9 | Audit & traceability | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | High |
| C10 | Limits & kill switch | L1 | L1 | L0 | L1 | 0.20 | — | **0.20** | High |

Controls where a risk surface exists: 2.80 / 8.0 (35%); 2 criteria scored SA (surface absent).

A small, read-mostly design server: it runs no code, talks only to Figma's API, and validates its arguments tightly, so a misused call mostly reads designs or writes image files. Its weak spots are around that core: Figma text comes back unmarked as part of the design data, the image tool writes and overwrites files with no explicit destructive label and no limits on how much it downloads, and there is no audit log. It also auto-loads a .env file from its working directory with no trust decision, and usage telemetry is on by default.

## Critical gaps
- A .env file in the server's working directory is auto-loaded with no trust decision and can set the Figma credentials, the proxy, the image directory and the tool set. (ASI06, ASI04, T1; C6) — [src/config.ts:88-89](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L88-L89); [src/config.ts:167](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L167)

## Criterion details

### C1 Identity & least privilege — 0.50 (high)

The server holds one credential, a Figma personal access token or OAuth token, and attaches it only to requests to Figma's fixed API address; image files are fetched from the URLs Figma returns without the token. Its tools only read from Figma, so through this server the token can read designs but not change them. All tools share that one long-lived token and there is no per-request authorization. The token can also come from a .env file in the working directory, which is loaded without any trust decision.

- **S L2:** A single dedicated Figma token (PAT via X-Figma-Token or OAuth Bearer) from a CLI flag or environment is used for every Figma API call. — [src/config.ts:98-101](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L98-L101); [src/services/figma.ts:38-51](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/services/figma.ts#L38-L51) (verified)
  - *To reach the next level:* No per-tool or read/write credential split; one long-lived token serves every call.
- **C L2:** Every Figma API request goes through FigmaService.getAuthHeaders to the hard-coded api.figma.com base; image downloads fetch Figma-provided URLs without credentials and there are no subprocesses. — [src/services/figma.ts:30](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/services/figma.ts#L30); [src/services/figma.ts:81-86](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/services/figma.ts#L81-L86); [src/utils/common.ts:33-35](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/utils/common.ts#L33-L35) (verified)
  - *To reach the next level:* No authorization layer checks individual tool calls; anything the host sends runs with the token.
- **D L1:** The token is not limited to the operator-supplied source: a .env file in the working directory is auto-loaded at startup and can supply the Figma credentials. — [src/config.ts:88-89](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L88-L89); [src/config.ts:139-145](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L139-L145) (verified)
  - *To reach the next level:* Credentials should come only from the host-supplied flag or environment, never from the working directory.
- **B L3:** Through this server a hijacked identity can only issue GET reads of files, nodes and image renders on api.figma.com; nothing is written to Figma or any other system. — [src/services/figma.ts:313-316](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/services/figma.ts#L313-L316); [src/services/figma.ts:30](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/services/figma.ts#L30) (verified)
  - *To reach the next level:* The token is long-lived and not task-scoped or short-lived.
- **Cap:** none

### C2 Approval gates — 0.30 (high)

As a tool server it relies on the MCP host to ask the user before calls. The data tool is correctly labelled read-only, but the image tool, which creates directories and writes or overwrites files in the image directory, carries only an open-world label and no explicit destructive or read-only flag, and there is no preview or dry run. Operators can drop the image tool with a flag, but it is on by default. A wrongly approved call can overwrite image files in the project, which is usually recoverable from version control.

- **S L1:** get_figma_data is annotated readOnlyHint: true; download_figma_images has only openWorldHint and no explicit readOnlyHint/destructiveHint despite writing and overwriting files. — [src/mcp/index.ts:86](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/mcp/index.ts#L86); [src/mcp/index.ts:100-107](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/mcp/index.ts#L100-L107) (verified)
  - *To reach the next level:* The write tool needs explicit readOnlyHint: false and an accurate destructiveHint/idempotentHint.
- **C L1:** The server adds no confirmation step on any path; the write tool's file effects are signalled only through absent hints. — [src/mcp/index.ts:100-107](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/mcp/index.ts#L100-L107); [src/utils/common.ts:44](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/utils/common.ts#L44) (verified)
  - *To reach the next level:* Every consequential path should be explicitly flagged, with a confirmation step or preview for overwrites.
- **D L1:** The image tool is registered by default; --skip-image-downloads or SKIP_IMAGE_DOWNLOADS removes it, but the environment form can also be set from the auto-loaded working-directory .env. — [src/mcp/index.ts:100](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/mcp/index.ts#L100); [src/config.ts:152-156](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L152-L156) (verified)
  - *To reach the next level:* A read-only mode on by default (write tool opt-in) is missing.
- **B L2:** A wrongly approved call writes or overwrites .png/.svg/.gif files inside the image directory (default: the server's working directory); there is no undo, but project files are usually under version control. — [src/utils/common.ts:44](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/utils/common.ts#L44); [src/config.ts:157-162](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L157-L162) (verified)
  - *To reach the next level:* No preview, overwrite protection or bound on the number of files a call can write.
- **Cap:** none

### C3 Tool & action scoping — 0.53 (high)

Arguments are checked in code: file keys must be alphanumeric, node IDs must match Figma's ID format, file names are restricted to safe characters with a .png, .svg or .gif extension, and the target directory must resolve inside the configured image directory. All requests go to Figma's fixed API address. The directory check is lexical, and the code notes it does not account for symlinks. There are no upper bounds on how many images a call downloads, on the PNG export scale or on traversal depth, and both tools are on by default.

- **S L2:** Typed zod schemas with allowlist regexes for fileKey, nodeId, fileName and filenameSuffix, plus lexical path containment for localPath; containment is not realpath-based (documented in the resolver). — [src/mcp/tools/download-figma-images-tool.ts:42-47](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/mcp/tools/download-figma-images-tool.ts#L42-L47); [src/utils/local-path.ts:38-41](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/utils/local-path.ts#L38-L41); [src/utils/local-path.ts:64-71](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/utils/local-path.ts#L64-L71) (verified)
  - *To reach the next level:* Containment should be checked on resolved real paths, and numeric/quantity bounds (nodes count, pngScale, depth) enforced.
- **C L3:** Both tools validate through their zod schemas (enforced by McpServer.registerTool), and the download path re-checks containment for each file before writing. — [src/mcp/tools/get-figma-data-tool.ts:15-17](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/mcp/tools/get-figma-data-tool.ts#L15-L17); [src/mcp/tools/download-figma-images-tool.ts:111](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/mcp/tools/download-figma-images-tool.ts#L111); [src/utils/common.ts:25-30](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/utils/common.ts#L25-L30) (verified)
  - *To reach the next level:* No central policy layer beyond per-tool schemas; quantity limits are absent from every tool.
- **D L1:** Both tools, including the file-writing image tool, are on by default; the write tool can be disabled individually. — [src/mcp/index.ts:100](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/mcp/index.ts#L100); [src/config.ts:152-156](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L152-L156) (verified)
  - *To reach the next level:* A read-only tool set by default with the write tool opt-in is missing.
- **B L2:** A misused tool reads any Figma file the token can access and writes image files anywhere inside the image directory, which defaults to the server's working directory. — [src/config.ts:157-162](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L157-L162); [src/mcp/tools/download-figma-images-tool.ts:74-80](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/mcp/tools/download-figma-images-tool.ts#L74-L80) (verified)
  - *To reach the next level:* No quantity bounds on images per call, scale or response size.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

The server never interprets model text as code: there is no shell, eval, subprocess or script execution in its runtime source. Its work is HTTPS requests to Figma, in-process image cropping and writing image files.

- **Structural absence:** searched `rg -n 'child_process|\bspawn\(|\beval\(|new Function|\bvm\.'` in `src` → 2 hits (Both hits are in src/tests/server.test.ts (the test harness spawning the server); no runtime code executes commands or evaluates code.)

### C5 Untrusted input blast radius — 0.17 (high)

The design data the server returns includes text, layer names and descriptions written by whoever authored the Figma file, which may be a third party. It is returned as one block of serialized text with no marker separating that content from the server's own structure and no untrusted flag. The server offers no egress of its own beyond Figma's API, so a hijacked host cannot use it to send data to an arbitrary address, but it can read other Figma files the token can access and write image files into the project without the server asking anyone.

- **S L1:** Design data, including author-controlled text, is returned as a single plain text block (tree/yaml/json serialization) with no provenance or untrusted marking. — [src/mcp/tools/get-figma-data-tool.ts:96](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/mcp/tools/get-figma-data-tool.ts#L96); [src/mcp/tools/get-figma-data-tool.ts:31-35](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/mcp/tools/get-figma-data-tool.ts#L31-L35) (verified)
  - *To reach the next level:* Outputs should separate author-supplied text from structural metadata and carry provenance.
- **C L0:** No source is distinguished; all Figma content enters the host's context the same way. — [src/mcp/tools/get-figma-data-tool.ts:96](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/mcp/tools/get-figma-data-tool.ts#L96) (verified)
  - *To reach the next level:* Untrusted Figma text should be marked on every output path.
- **D L0:** No provenance or isolation mechanism exists, and the mode that drops the write leg (--skip-image-downloads) is off by default. — [src/config.ts:152-156](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L152-L156) (verified)
  - *To reach the next level:* A default mode without the write tool, or with marked content, is missing.
- **B L2:** A hijacked host can read other Figma files the token can access and write or overwrite image files in the image directory unattended, but the server has no outbound channel beyond Figma's own API. — [src/services/figma.ts:30](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/services/figma.ts#L30); [src/utils/common.ts:44](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/utils/common.ts#L44) (verified)
  - *To reach the next level:* Writes should require confirmation or be confined to new files once untrusted content has been read.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.25 (high)

The server keeps no memory and reads no instruction files. At startup, though, it loads a .env file from its working directory, which may be the user's project or a cloned repository, with no trust decision. That file can set the Figma credentials, the outbound proxy, the image directory, whether the image tool is offered and the HTTP host and port, and its settings apply to every launch from that directory.

- **S L0:** A .env in the working directory is auto-loaded and feeds security-relevant settings (Figma token, FIGMA_PROXY, IMAGE_DIR, SKIP_IMAGE_DOWNLOADS, host and port) with no trust prompt. — [src/config.ts:139-142](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L139-L142); [src/config.ts:88-89](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L88-L89); [src/config.ts:167](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L167) (verified)
  - *To reach the next level:* Configuration should be read only from the host environment or an explicit --env path.
- **C L0:** No control applies to any of the settings read from that file. — [src/config.ts:148-162](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L148-L162) (verified)
  - *To reach the next level:* All security-relevant settings need protection from workspace-supplied values.
- **D SA:** No memory store or per-user namespace exists; there is nothing to isolate across users. — searched `rg -n 'memory|AGENTS\.md|CLAUDE\.md|vector'` in `src` → 6 hits (All hits are comments about SVG vector shapes; no memory store, instruction-file loading or retrieval index exists.) (verified)
- **B L2:** A poisoned .env persists for every launch from that directory and can change the credential, proxy and image directory and re-enable the image tool, but cannot trigger tool calls on its own. — [src/config.ts:88-89](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L88-L89); [src/server.ts:45-51](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/server.ts#L45-L51) (verified)
  - *To reach the next level:* Workspace-supplied configuration should be ignored or surfaced for approval.
- **Cap:** C6-REPOCONFIG — A .env file in the server's working directory is loaded at startup with no trust decision and can set the Figma credentials, the outbound proxy, the image directory and the tool set.

### C7 Third-party extensions — 1.00 (high)

The server loads no plugins, launches no other MCP servers and installs nothing at runtime; the only dynamic imports load its own modules and Node built-ins. (Hosts launching it with npx -y is the host's supply-chain choice, not something this server does.)

- **Structural absence:** searched `rg -n 'import\('` in `src` → 6 hits (Four hits in src/utils/image-processing.ts import the server's own logger/common modules and node:fs/promises; two are test mocks. No third-party code is loaded at runtime.); searched `rg -n 'child_process|\bspawn\(|\beval\(|new Function|\bvm\.'` in `src` → 2 hits (Test harness only; nothing is launched at runtime.)

### C8 Secrets & sensitive-data protection — 0.47 (high)

The Figma token is kept out of the places it does not need to be: startup output masks it, error bodies returned from Figma are scrubbed of it before reaching the model, and telemetry error messages are scrubbed of both startup and per-request tokens. Usage telemetry to PostHog is on by default; it sends metrics rather than design content, but error messages (after token scrubbing) can include Figma file identifiers and local paths. The README shows the token passed as a command-line argument in the host's config file, and the token is long-lived.

- **S L2:** Secrets come from flag/env; a masking helper covers startup output and redaction helpers scrub tokens from Figma error bodies and telemetry error messages. — [src/config.ts:83-86](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L83-L86); [src/utils/fetch-json.ts:137-138](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/utils/fetch-json.ts#L137-L138); [src/telemetry/client.ts:152](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/telemetry/client.ts#L152) (verified)
  - *To reach the next level:* No OS keychain or secret manager; tokens are kept in plaintext host configuration.
- **C L3:** Redaction applies on the model-bound error path (Figma response bodies), telemetry (init-time and per-request secrets) and startup logs; there are no subprocesses or transcripts. — [src/services/figma.ts:83-86](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/services/figma.ts#L83-L86); [src/server.ts:53-56](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/server.ts#L53-L56); [src/server.ts:152](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/server.ts#L152) (verified)
  - *To reach the next level:* Telemetry error text is not minimised beyond token scrubbing (file identifiers and paths still pass).
- **D L1:** Telemetry to PostHog is on by default (opt out with --no-telemetry, FRAMELINK_TELEMETRY=off or DO_NOT_TRACK) with GeoIP enabled; it is mostly metrics but includes error messages. — [src/telemetry/client.ts:81-86](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/telemetry/client.ts#L81-L86); [src/telemetry/client.ts:117](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/telemetry/client.ts#L117); [src/telemetry/capture.ts:38](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/telemetry/capture.ts#L38) (verified)
  - *To reach the next level:* Telemetry should be opt-in, or strictly content-free by default.
- **B L1:** A leaked token is long-lived and, as typically created for this use, reads the user's Figma files; the README places it on the command line in plaintext host config. — [src/config.ts:98-99](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/config.ts#L98-L99) (verified)
  - *To reach the next level:* Tokens are not short-lived or minimally scoped by guidance or code.
- **Cap:** none

### C9 Audit & traceability — 0.33 (high)

The server writes plain-text progress lines to stderr for each tool call, such as which file and node were fetched and where each image was saved, plus error lines. These are not structured, do not record all arguments or results consistently, and are not stored by the server; any lasting record depends on the host capturing stderr. Usage telemetry records each call's metrics in PostHog, but that is analytics, not an audit trail the operator controls.

- **S L1:** Unstructured Logger lines on stderr per call (file key, node, saved image paths, errors). — [src/mcp/tools/get-figma-data-tool.ts:58-62](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/mcp/tools/get-figma-data-tool.ts#L58-L62); [src/utils/image-processing.ts:156](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/utils/image-processing.ts#L156); [src/utils/logger.ts:4-14](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/utils/logger.ts#L4-L14) (verified)
  - *To reach the next level:* A structured per-call record (tool, arguments, status, timestamp) is missing.
- **C L2:** Both tools log their calls and errors; validation rejects are reported only to telemetry. — [src/mcp/tools/download-figma-images-tool.ts:192](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/mcp/tools/download-figma-images-tool.ts#L192); [src/mcp/tools/get-figma-data-tool.ts:100](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/mcp/tools/get-figma-data-tool.ts#L100) (verified)
  - *To reach the next level:* Rejected calls and configuration changes are not recorded locally.
- **D L1:** Logging is on by default but goes only to stderr; the server persists nothing itself. — [src/utils/logger.ts:4-14](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/utils/logger.ts#L4-L14) (verified)
  - *To reach the next level:* A persisted record outside the workspace, written by default, is missing.
- **B L1:** Records are best-effort console lines that are lost unless the host captures them. — [src/utils/logger.ts:4-14](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/utils/logger.ts#L4-L14) (verified)
  - *To reach the next level:* Records need to be durable per action.
- **Cap:** none

### C10 Limits & kill switch — 0.20 (high)

The server sets no timeout on Figma API calls or image downloads, does not cap how many images one call fetches (they all download in parallel), how large a response or file may be, or the PNG export scale, and fetches the whole file tree unless the model passes a depth. Only error bodies and telemetry messages are truncated. Cancellation from the host is not acted on; stopping the process ends everything because it is a single process, and Figma's own rate limits are the only outside ceiling.

- **S L1:** Only caller-chosen limits exist (optional depth); error-body truncation is the only server-side cap. — [src/mcp/tools/get-figma-data-tool.ts:31-35](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/mcp/tools/get-figma-data-tool.ts#L31-L35); [src/utils/fetch-json.ts:57](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/utils/fetch-json.ts#L57); [src/utils/fetch-json.ts:65](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/utils/fetch-json.ts#L65) (verified)
  - *To reach the next level:* Server-enforced timeouts and caps on image count, scale and response size are missing.
- **C L1:** The depth limit applies only to get_figma_data; downloads run all requested images concurrently with no bound. — [src/services/figma.ts:299-300](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/services/figma.ts#L299-L300) (verified)
  - *To reach the next level:* Bounds and timeouts are missing on the download path and every Figma request.
- **D L0:** With no depth argument the full file is fetched and simplified; no default limits exist. — [src/services/figma.ts:313](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/services/figma.ts#L313) (verified)
  - *To reach the next level:* Sensible default caps are missing.
- **B L1:** A runaway call is bounded only by Figma's rate limits and the host; stopping the process (SIGINT/SIGTERM) exits immediately. — [src/server.ts:110-126](https://github.com/GLips/Figma-Context-MCP/blob/c083d65c7e002923e7cb98f4e3bdafb105e90f6d/src/server.ts#L110-L126) (verified)
  - *To reach the next level:* No per-call time or size ceilings, and host cancellation does not abort in-flight requests.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Author-controlled Figma text and layer names returned as design data (src/mcp/tools/get-figma-data-tool.ts:96) · [B] sensitive data/systems: Token reads any Figma file the user can access (src/services/figma.ts:30, src/services/figma.ts:38-51) · [C] state change / egress: download_figma_images writes/overwrites files in the image directory (src/utils/common.ts:44); no egress beyond api.figma.com · Same default session? Yes

## Highest-impact improvements
1. Stop auto-loading .env from the working directory; read configuration only from the host environment or an explicit --env path. (C6 S L0→L3, +0.225 before caps; Playbook 2)
2. Annotate download_figma_images with readOnlyHint: false and an accurate destructiveHint, and refuse to overwrite existing files unless asked. (C2 S L1→L2, +0.075 before caps; Playbook 5)
3. Return design data as structured content that separates author-supplied text from structure and flags it as untrusted. (C5 S L1→L2, +0.075 before caps; Playbook 1)
4. Add request timeouts and server-side caps on images per call, PNG scale, depth and response/file size. (C10 S L1→L2, +0.075 before caps; Playbook 3 step 3)
5. Log each tool call (tool, arguments, status, timestamp) as structured JSON to stderr. (C9 S L1→L2, +0.075 before caps; Playbook 1 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the README's stdio configuration. Running the binary without --stdio starts a Streamable HTTP server on 127.0.0.1:3333 with no client authentication of its own, using the server's configured token; that mode was not scored.
- HEAD is one commit after tag v0.13.2 (package.json says 0.13.2), so no release version is recorded.
- MCP SDK 1.29.0 behaviour (zod validation in McpServer.registerTool, host-header checks in createMcpExpressApp) and undici proxy behaviour were inferred from library knowledge, not read in the dependency.
- Figma token scopes and expiry are set by the user in Figma and were not examined; the C1 and C8 blast-radius ratings assume a typical read-capable personal access token.
- No reviewer-steering text was found in the repository (CLAUDE.md contains only contributor guidance).
