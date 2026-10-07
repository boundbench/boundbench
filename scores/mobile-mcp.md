# Defense-in-depth score: Mobile MCP

**Repo:** https://github.com/mobile-next/mobile-mcp · **Commit:** `2c6413c84ddcb8aa1760484a0e467b7aa43551d4` · **Reviewed:** 2026-10-05
**What it is:** MCP server that lets an AI agent drive real and simulated iOS and Android devices: list and launch apps, tap, swipe, type, read the screen, take screenshots and use a remote device fleet.
**Category:** AI Assistants
**Scored configuration:** stdio MCP server as launched by the shipped `npx -y @mobilenext/mobile-mcp@latest` config, no environment variables set (default mobilecli robot, telemetry on, no LOG_FILE).
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 3.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | none | **0.05** | High |
| C2 | Approval gates | L1 | L2 | L2 | L1 | 0.38 | none | **0.38** | High |
| C3 | Tool & action scoping | L2 | L2 | L0 | L1 | 0.35 | none | **0.35** | High |
| C4 | Code-execution isolation | L1 | L2 | L2 | L0 | 0.33 | none | **0.33** | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C7 | Third-party extensions | L1 | L0 | L0 | L1 | 0.12 | none | **0.12** | Medium |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L0 | L1 | 0.20 | none | **0.20** | Medium |
| C9 | Audit & traceability | L1 | L2 | L0 | L2 | 0.33 | G1 | **0.33** | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | none | **0.33** | High |

Controls where a risk surface exists: 2.07 / 9.0 (23%); 1 criterion scored SA (surface absent).

Mobile MCP gives an agent full control of any phone, emulator or simulator connected to the machine, plus a billed remote device fleet, with every tool enabled and no approval, read-only mode or device allowlist of its own. Screen text, clipboard and device logs come back to the model as plain text, so a hijacked agent can read private data on the device and type, send or uninstall on it without the server asking anyone. Command handling is careful (no host shell, typed arguments, an extension and directory allowlist on file writes), but logging records full tool payloads on by default and limits on long-running or billed work are thin.

## Critical gaps
- Worst case under prompt injection from device content: default tools let a hijacked agent read the clipboard and screen, send data out through the device browser or apps, and uninstall apps or send messages, with no server-side human check (C5-WORSTCASE). (ASI01, LLM01, T6; C5). Evidence: [src/server.ts:759-763](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L759-L763); [src/server.ts:791-799](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L791-L799); [src/server.ts:1058-1059](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L1058-L1059)
- Model-chosen app bundles from the host's disk are installed and launched with no server-side restriction; on an iOS Simulator they run with the logged-in user's host access. (ASI05, T11, LLM05; C4). Evidence: [src/server.ts:603-621](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L603-L621); [src/server.ts:570-585](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L570-L585)

## Criterion details

### C1 Identity & least privilege: 0.05 (high confidence)

The server holds no API keys of its own and does not forward client tokens, but it does nothing to narrow the authority it runs with. Every tool reaches whichever device the model names among all phones, emulators and simulators visible on the machine, and the remote-fleet tools act on the device-cloud account the user logged into, which is billed. Helper processes inherit the server's full environment. There is no device allowlist, read-only identity or per-request authorization, so a misused session can act on a personal phone with all of its signed-in apps.

- **S L0:** All device control runs through the mobilecli binary with the OS user's ambient authority and inherited environment; nothing scopes which device or account a call may use. Evidence: [src/mobilecli.ts:84-92](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/mobilecli.ts#L84-L92); [src/server.ts:293-305](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L293-L305) (verified)
  - *To reach the next level:* No dedicated or scoped identity: no per-device allowlist, read-only mode, or narrowed fleet credential.
- **C L0:** Each device tool builds a mobilecli invocation for the model-supplied device id with no authorization step in between. Evidence: [src/mobile-device.ts:114-117](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/mobile-device.ts#L114-L117) (verified)
  - *To reach the next level:* No authorization layer that every tool path passes through.
- **D L0:** By default the server accepts any online device the host can see, including real personal devices, and exposes the remote-fleet allocation tools. Evidence: [src/server.ts:285-291](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L285-L291); [src/server.ts:495-517](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L495-L517) (verified)
  - *To reach the next level:* No narrower default, such as limiting the server to an operator-named device or to simulators and emulators.
- **B L1:** A hijacked session can act with write access in every app signed in on a connected device, read its clipboard, and reserve billed remote devices; it gets no host credentials directly. Evidence: [src/server.ts:1043-1064](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L1043-L1064); [src/server.ts:498-499](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L498-L499) (verified)
  - *To reach the next level:* Authority spans many systems (every account on the device plus the fleet account) rather than one scoped project.
- **Cap:** none

### C2 Approval gates: 0.38 (high confidence)

As a tool server, Mobile MCP leaves approval to the MCP host and does not ask anyone itself. Every tool is registered with read-only, destructive and open-world hints, and the batch tool is flagged destructive, which gives a host something to gate on. The hints are not accurate on every state-changing tool (typing, tapping and installing apps are not marked destructive, though they can send messages or change data, and the device-log tool is marked read-only even though it can save a file), and there is no preview, dry-run or server-side read-only mode. Uninstalling apps, releasing a remote device and sending typed text cannot be undone.

- **S L1:** Annotations exist on every tool, but some state-changing tools such as text entry with submit, taps and app install are marked non-destructive, and the device-log tool is marked read-only although it can write a file to the host. Evidence: [src/server.ts:832-850](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L832-L850); [src/server.ts:603-611](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L603-L611); [src/server.ts:1075-1077](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L1075-L1077) (verified)
  - *To reach the next level:* Accurate read-only and destructive hints on every state-changing tool (L2), then a preview or dry-run for destructive operations.
- **C L2:** One registration helper attaches annotations to every tool, and the batch tool that runs other tools' callbacks directly is itself flagged destructive with its full step list visible to the host. Evidence: [src/server.ts:137-143](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L137-L143); [src/server.ts:1254](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L1254) (verified)
  - *To reach the next level:* Steps inside a batch are not surfaced to the host as individual calls, so a host gate sees one coarse destructive call.
- **D L2:** The hints are fixed in source and cannot be changed by the model or by content, but the server offers no read-only or confirmation mode an operator could turn on. Evidence: [src/server.ts:137-143](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L137-L143) (verified)
  - *To reach the next level:* No server-enforced read-only mode or confirmation step.
- **B L1:** Wrongly approved calls can uninstall apps with their data, wipe a remote device on release, and send typed text inside any app; only orientation, location and similar settings are easily reversible. Evidence: [src/server.ts:624-637](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L624-L637); [src/server.ts:522-523](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L522-L523); [src/server.ts:845-847](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L845-L847) (verified)
  - *To reach the next level:* No checkpoint, undo or preview for destructive device actions.
- **Cap:** none

### C3 Tool & action scoping: 0.35 (high confidence)

Tools are fairly narrow device actions with typed arguments: coordinates, durations, locations and log limits are bounded, enums restrict directions and orientations, and batch steps are re-validated against each tool's schema. Paths the server writes to are checked against an extension list and a working-or-temp-directory allowlist. Other arguments are looser: the URL tool checks only the scheme and can open any host, the install tool accepts an app file from anywhere on the host, and package names and element refs are passed through as given. Every tool, including install, uninstall and remote-device allocation, is enabled by default with no way to select a smaller set.

- **S L2:** Typed schemas with numeric bounds, an extension allowlist and a directory allowlist on writes, and a scheme-only check on URLs. Evidence: [src/server.ts:791-795](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L791-L795); [src/server.ts:612-616](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L612-L616); [src/utils.ts:32-35](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/utils.ts#L32-L35); [src/server.ts:1017-1021](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L1017-L1021) (verified)
  - *To reach the next level:* URL opening is not host-restricted, app-install paths are unrestricted, and package names and refs are not validated on the default path.
- **C L2:** Most tools validate through their zod schemas, and the batch tool re-parses each step with the target tool's schema. Evidence: [src/server.ts:1276](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L1276); [src/mobile-device.ts:226-249](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/mobile-device.ts#L226-L249) (verified)
  - *To reach the next level:* No shared policy layer covering free-form arguments (URLs, package names, refs, install paths) across all tools.
- **D L0:** All tools, including install, uninstall, URL opening and billed remote-device allocation, are registered unconditionally. Evidence: searched `rg -n -S -e 'enabled_tools|toolsets|READ_ONLY|readonly|--tools'` in `src/` → 3 hits (all three hits are TypeScript `private readonly` constructor parameters; no tool selection or read-only switch exists) (verified)
  - *To reach the next level:* No read-only default tool set or tool-group selection.
- **B L1:** A misused tool set gives general control of any connected device and any web host through its browser, with only minor limits on host file writes. Evidence: [src/server.ts:831-850](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L831-L850) (verified)
  - *To reach the next level:* Device actions are not scoped to a test device or bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation: 0.33 (medium confidence)

The server never hands model text to a host shell: every helper (mobilecli, and adb, go-ios and simctl in legacy mode) is started with a fixed binary and an argument list. Its code-execution surface is on the device: the model can install any app bundle it can name from the host's disk and then launch it, and the server installs its own helper app on iOS Simulators at first use. The only boundary around that code is the device itself. A real phone or an Android emulator is a separate system, but iOS Simulator apps run as ordinary processes of the logged-in macOS user, so an untrusted app bundle installed there runs with the user's own access to files and network. The server has no option to limit which kinds of device or which app files may be used.

- **S L1:** Model-chosen app bundles are installed and launched with no isolation provided by the server; the only separation is the target device, which for the iOS Simulator is a same-user host process (inferred from how the simulator runs apps). Evidence: [src/server.ts:603-621](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L603-L621); [src/server.ts:296-300](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L296-L300) (inferred)
  - *To reach the next level:* No server-enforced boundary, such as restricting installs to emulators or VMs, or to signed or allowlisted app files.
- **C L2:** Every host-side helper runs as a fixed binary with an argv array, so model text reaches code execution only by way of the device; host-side exceptions (legacy unzip and plist conversion) are low-power. Evidence: [src/mobilecli.ts:84-108](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/mobilecli.ts#L84-L108); searched `rg -n -S -e 'shell:\s*true|execSync\(|\beval\(|new Function'` in `src/` → 0 hits (no shell-mode exec, eval or Function constructor anywhere in the server) (verified)
  - *To reach the next level:* The device boundary is the same for every path but is not itself a sandbox the server controls.
- **D L2:** The model may pick any connected device, including an iOS Simulator, and an operator environment variable switches Android and physical iOS to the legacy adb and go-ios path without further warning. Evidence: [src/server.ts:266](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L266); [src/server.ts:293-305](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L293-L305) (verified)
  - *To reach the next level:* No operator control over which device types may receive installed code, and no per-call human escalation for installs.
- **B L0:** An app bundle installed and launched on an iOS Simulator runs as the logged-in macOS user with that user's files, network and credentials (inferred from the simulator's process model; the server adds no restriction). Evidence: [src/server.ts:603-611](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L603-L611); [src/server.ts:570-585](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L570-L585) (inferred)
  - *To reach the next level:* Installed code is not confined away from the host user's home directory, network and credentials.
- **Cap:** none

### C5 Untrusted input blast radius: 0.00 (high confidence)

Everything the agent reads through this server comes from the device: on-screen text and labels from any app or web page, the clipboard, device logs and crash reports. All of it is returned to the model as plain text with no marker saying it is untrusted, and some server outputs carry instructions to the model alongside the data. Nothing in the server limits what a session can do after reading such content. If a message, web page or notification on the device hijacks the agent, the default tools let it read private data (clipboard, messages on screen) and send it out by opening a URL on the device browser or typing it into an app, and uninstall apps or send messages, with no human asked by the server.

- **S L0:** Tool outputs mix server directives to the model (for example the no-device hint telling the model what to say and which tool to call) with returned data, and device content has no provenance. Evidence: [src/server.ts:33-35](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L33-L35); [src/server.ts:51-58](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L51-L58) (verified)
  - *To reach the next level:* Structured outputs that separate returned device content from server metadata, with an untrusted flag.
- **C L0:** Screen elements, clipboard text, device logs and crash reports are all returned as plain text with the same standing as the server's own messages. Evidence: [src/server.ts:759-763](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L759-L763); [src/server.ts:1058-1059](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L1058-L1059); [src/server.ts:1231-1234](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L1231-L1234) (verified)
  - *To reach the next level:* Any untrusted source distinguished from the server's own text.
- **D L0:** No untrusted-content handling exists to be on by default; the only matching hit is an escaping comment in the legacy Android path. Evidence: searched `rg -n -S -e 'untrusted|provenance|taint|injection'` in `src/` → 1 hits (single hit is a shell-escaping comment in src/android.ts, not an untrusted-content control); [src/server.ts:144-156](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L144-L156) (verified)
  - *To reach the next level:* A provenance or no-egress mode that is on by default.
- **B L0:** A hijacked agent can read the clipboard and screen, exfiltrate by opening any http(s) URL on the device or typing into a messaging app, and uninstall apps or send messages, with no server-side human check. Evidence: [src/server.ts:791-799](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L791-L799); [src/server.ts:1058-1059](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L1058-L1059); [src/server.ts:633-635](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L633-L635) (verified)
  - *To reach the next level:* Exfiltration and irreversible device actions would need to require human approval once device content has been read.
- **Cap:** C5-WORSTCASE: Under the default tool set a hijacked agent can both read and send out device data and take irreversible device actions without any server-side human check.

### C6 Memory, context & configuration integrity: 1.00 (high confidence)

The server has no memory store, does not load .env files, instruction files or project configuration from the working directory, and reads nothing it wrote back into the model's context. Its only configuration is environment variables set by whoever launches it. The files it can write (screenshots, logs, recordings) are limited to image, log, text and video extensions, so it cannot create instruction or settings files that other tools auto-load. Device state such as installed apps persists on the device, but that is the device, not the agent's memory.

- **Structural absence:** searched `rg -n -S -e 'dotenv|memory|remember|persist|AGENTS\.md|CLAUDE\.md|\.mcp\.json|settings\.json'` in `src/` → 0 hits (no memory, dotenv, instruction-file or project-config loading); searched `rg -n -S -e 'readFileSync|readdirSync|createReadStream'` in `src/` → 4 hits (reads are the MCP SDK's package.json (version string), a legacy iOS temp screenshot, and listing a temp unzip directory; none feed persisted content back to the model)

### C7 Third-party extensions: 0.12 (medium confidence)

The server loads no plugins and connects to no other MCP servers. The third-party code it handles is app packages: the model can install any .apk, .ipa, .zip or .app file it names from the host and launch it, with no signature, hash or allowlist check and no consent step in the server, and the server silently installs its own helper agent onto iOS Simulators the first time it uses one. Installed apps run as separate processes on the device; on an iOS Simulator that means the logged-in user's account on the host. The shipped launch configs fetch the latest published server package on every start, which is the project's own distribution rather than a runtime extension and is not scored here.

- **S L1:** App files chosen by the model are installed from any host path with only an extension check; nothing pins or verifies them. Evidence: [src/server.ts:612-620](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L612-L620) (verified)
  - *To reach the next level:* No pinning, hash or signature check, or allowlist of app files the server will install.
- **C L0:** Neither model-chosen app installs nor the automatic simulator agent install are verified by the server. Evidence: searched `rg -n -S -e 'sha256|digest|checksum|signature|integrity|codesign'` in `src/` → 3 hits (hits are the PNG magic-byte check and the telemetry distinct-id hash; nothing verifies installed code) (verified)
  - *To reach the next level:* Verification of at least one kind of installed code.
- **D L0:** The helper agent is installed onto an iOS Simulator automatically when its status check fails, and model-requested installs proceed with no consent step in the server. Evidence: [src/server.ts:296-300](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L296-L300) (verified)
  - *To reach the next level:* No explicit consent showing the exact package before third-party code is installed.
- **B L1:** An installed app runs as a separate process; on an iOS Simulator that is the same host user with that user's environment (inferred from the simulator's process model). Evidence: [src/server.ts:570-585](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L570-L585) (inferred)
  - *To reach the next level:* Installed code is not sandboxed away from the host user's files and credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection: 0.20 (medium confidence)

The server keeps no secrets of its own beyond an optional HTTP bearer token read from the environment, and remote-fleet login is handled by the separate mobilecli tool. Usage telemetry to PostHog (plus a Scarf pixel) is on by default; it is content-free, sending tool names, durations, device counts, the client name and a hashed host id, and can be turned off with an environment variable. However, every tool call's full arguments and full text result are written to stderr by default, and to a file when LOG_FILE is set, with no redaction, so text typed into apps (including passwords), clipboard contents and device logs end up in the MCP host's logs. Clipboard and device-log contents also go to the model unfiltered.

- **S L1:** The only secret is read from an environment variable, and telemetry hashes the host identity; there is no masking or redaction anywhere. Evidence: [src/index.ts:33](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/index.ts#L33); [src/server.ts:183-184](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L183-L184) (verified)
  - *To reach the next level:* Redaction of typed text, clipboard and log content on logging paths.
- **C L1:** Only the telemetry path is minimised; logs, model-bound results, error messages and subprocess environments are not. Evidence: [src/server.ts:185-192](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L185-L192); [src/server.ts:147-152](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L147-L152) (verified)
  - *To reach the next level:* Protection on logs and transcripts as well as telemetry.
- **D L0:** Full payload logging of every call's arguments and results to stderr is always on, and content-free telemetry is on by default. Evidence: [src/logger.ts:3-13](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/logger.ts#L3-L13); [src/server.ts:175-178](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L175-L178); [src/server.ts:234](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L234) (verified)
  - *To reach the next level:* Payload logging off or redacted by default.
- **B L1:** The server holds no long-lived keys itself, but the model can read whatever is on the device clipboard, screen and logs, which commonly include passwords, one-time codes and app tokens (inferred). Evidence: [src/server.ts:1043-1059](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L1043-L1059) (inferred)
  - *To reach the next level:* Device secrets are not kept out of model-bound results or logs.
- **Cap:** none

### C9 Audit & traceability: 0.33 (high confidence)

Every tool registered through the server's helper writes a plain-text line with the tool name and full arguments before it runs, and another with the full result afterwards. By default these lines only go to stderr, where whether they are kept depends on the MCP host; a durable file with timestamps exists only when the operator sets LOG_FILE. The record is unstructured, has no actor, session or request identifiers, and the steps inside a batch are not recorded one by one (only the batch's arguments). When the file is enabled it is appended synchronously before each action, but it sits wherever the operator points it with no tamper protection.

- **S L1:** Unstructured text lines of tool name, JSON arguments and result text; the screenshot tool logs only the image size. Evidence: [src/server.ts:147-152](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L147-L152); [src/server.ts:939](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L939) (verified)
  - *To reach the next level:* A structured per-call record with timestamps and result status in the default output.
- **C L2:** All tools registered through the shared helper are logged, including the batch tool's full step list. Evidence: [src/server.ts:137-152](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L137-L152); [src/server.ts:1276-1278](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L1276-L1278) (verified)
  - *To reach the next level:* Steps run inside a batch call their callbacks directly and are not recorded individually.
- **D L0:** The server's only durable record is the LOG_FILE append, which is off unless the operator sets the variable; otherwise lines go to stderr for the host to keep or drop. Evidence: [src/logger.ts:4-9](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/logger.ts#L4-L9) (verified)
  - *To reach the next level:* A durable record on by default, outside anything the server's own tools can write.
- **B L2:** When enabled, each line is appended synchronously before the tool runs, so records are flushed per action and a write error surfaces as a tool error. Evidence: [src/logger.ts:9](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/logger.ts#L9) (verified)
  - *To reach the next level:* The trajectory cannot be fully replayed (batch steps and screenshots are summarised).
- **Cap:** G1: The only durable audit record (LOG_FILE) is opt-in.

### C10 Limits & kill switch: 0.33 (high confidence)

Some operations have hard bounds: screenshot and crash-report commands time out after 30 seconds with an 8 MB output cap, log capture stops after at most 10,000 entries or 30 seconds of silence, and the login prompt times out after 15 seconds. Most device commands, though, run with no timeout, a batch can contain any number of steps, and the model chooses how long to wait for a billed remote device and how long a screen recording runs, with no maximum. Screen recordings and the login flow run as background processes, and when the server is stopped it simply exits: background processes are not killed and remote devices stay reserved until released.

- **S L2:** Server-enforced timeouts and output caps exist on screenshot, crash-report, log-capture and login operations. Evidence: [src/mobilecli.ts:101-108](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/mobilecli.ts#L101-L108); [src/server.ts:25-26](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L25-L26) (verified)
  - *To reach the next level:* Caps on every operation plus concurrency or rate limits.
- **C L1:** The general command runner used by most device tools has no timeout, and batch length is unbounded. Evidence: [src/mobilecli.ts:84-92](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/mobilecli.ts#L84-L92); [src/server.ts:1247-1250](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L1247-L1250) (verified)
  - *To reach the next level:* A timeout on every device command and a cap on batch size.
- **D L1:** The model sets the remote-allocation wait and the recording length with no upper bound. Evidence: [src/server.ts:509](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L509); [src/server.ts:1122](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L1122) (verified)
  - *To reach the next level:* Sensible maximums on model-chosen waits and durations.
- **B L1:** On SIGINT or SIGTERM the server exits without stopping recording or login child processes or releasing billed remote devices. Evidence: [src/index.ts:98-103](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/index.ts#L98-L103); [src/server.ts:1137-1144](https://github.com/mobile-next/mobile-mcp/blob/2c6413c84ddcb8aa1760484a0e467b7aa43551d4/src/server.ts#L1137-L1144) (verified)
  - *To reach the next level:* Stopping the server does not cancel background work or release reserved devices.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: On-screen text from any app or web page, clipboard, device logs and crash reports returned to the model (src/server.ts:761, src/server.ts:1059, src/server.ts:1234) · [B] sensitive data/systems: Clipboard and every app signed in on the connected device (src/server.ts:1059) · [C] state change / egress: Typing into apps, opening any http(s) URL on the device, uninstalling apps, reserving billed remote devices (src/server.ts:843, src/server.ts:798, src/server.ts:634, src/server.ts:514) · Same default session? Yes

## Highest-impact improvements
1. Add an operator-set device allowlist (or a default of emulators and simulators only) so a session cannot drive a personal phone unless explicitly enabled. (C1 D L0→L1, +0.050 before caps; Playbook 4)
2. Ship a read-only mode and tool groups, with install, uninstall, URL opening and remote allocation off by default. (C3 D L0→L2, +0.100 before caps; Playbook 3)
3. Stop logging full tool arguments and results to stderr by default; log tool names and status, and redact typed text and clipboard contents. (C8 D L0→L2, +0.100 before caps; Playbook 4)
4. Return device content as structured output marked untrusted, and move model-facing hints out of tool results. (C5 S L0→L2, +0.150 before caps; Playbook 1)
5. Put a timeout on every device command, cap batch size, allocation wait and recording length, and kill child processes on shutdown. (C10 C L1→L2, +0.075 before caps; Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- No release tag points at the pinned commit; package.json carries a placeholder version set at publish time.
- The mobilecli binary (a separate project) performs the actual device work, fleet login and helper-agent install; its behaviour was not reviewed and is taken only from how this server calls it.
- How iOS Simulator apps are confined on the host is inferred from the simulator's process model, not from code in this repository (C4 S and B, C7 B).
- The legacy robot path (MOBILEMCP_LEGACY_ROBOT=1), the optional Streamable HTTP mode (--listen, with optional MOBILEMCP_AUTH bearer token and a startup warning when unset), and the bundled Claude Code mirror plugin were read but not scored as the default.
- The shipped launch configs run `npx -y @mobilenext/mobile-mcp@latest`, so each start fetches the newest published package; this is the project's own distribution and is not scored under C7.
