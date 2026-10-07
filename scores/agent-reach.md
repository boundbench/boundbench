# Defense-in-depth score: Agent Reach

**Repo:** https://github.com/Panniantong/Agent-Reach · **Commit:** `a19a171fa980a0785849596492e0af4db800c82f` · **Reviewed:** 2026-10-05
**What it is:** Installer, health checker and skill that let AI coding agents read and search web pages, social platforms, video and GitHub through upstream CLIs.
**Category:** AI Assistants
**Scored configuration:** README-led flow: a host coding agent follows docs/install.md, runs `agent-reach install --env=auto --system` (core channels, Exa search via mcporter, skill copied into user skill folders) and then uses the skill through its shell; no optional channels or cookies.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents no · external communication yes

## Score: 1.9 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L1 | L1 | 0.25 | none | **0.25** | High |
| C2 | Approval gates | L1 | L0 | L1 | L1 | 0.17 | C2-SELFAPPROVE | **0.17** | High |
| C3 | Tool & action scoping | L2 | L1 | L2 | L0 | 0.33 | none | **0.33** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | L1 | L1 | L1 | L0 | 0.20 | none | **0.20** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L3 | L1 | 0.50 | none | **0.50** | High |
| C9 | Audit & traceability | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C10 | Limits & kill switch | L2 | L1 | L2 | L1 | 0.38 | none | **0.38** | High |


Agent Reach connects a coding agent to the open internet by installing third-party CLIs and telling the agent which commands to run, and its own code is careful with stored secrets and its own inputs. But it adds no safeguards around what the agent then does: untrusted web content, the user's GitHub login and arbitrary URL fetches end up in one session, with no approval step, isolation or audit trail from Agent Reach. Most installed tools are unpinned, and the search tool also reads server definitions from the workspace. Treat it as giving the agent the internet with only the host agent's own prompts as protection.

## Critical gaps
- The installer's only consent step is a --system flag the agent passes itself, and the skill's references list GitHub write commands with no gate. (ASI09, ASI02; C2). Evidence: [agent_reach/cli.py:270](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L270); [agent_reach/skill/references/dev.md:29](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/references/dev.md#L29)
- Every command the skill directs and every package the installer adds runs on the host as the user, with the full environment and network. (ASI05; C4). Evidence: [agent_reach/utils/process.py:16](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/utils/process.py#L16); [agent_reach/cli.py:1266](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L1266); [agent_reach/skill/references/web.md:35](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/references/web.md#L35)
- Untrusted web content, the user's GitHub login and arbitrary URL fetch share one session, so an injected page can leak data and act on the user's account. (ASI01, LLM01; C5). Evidence: [agent_reach/skill/SKILL_en.md:68](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/SKILL_en.md#L68); [agent_reach/skill/references/dev.md:29](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/references/dev.md#L29); [docs/README_en.md:115](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/docs/README_en.md#L115)
- The search command the skill runs also reads MCP server definitions from the workspace, so a cloned repository can change which server starts, with no trust prompt. (ASI06, ASI04; C6). Evidence: [agent_reach/channels/mcporter.py:37-41](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/channels/mcporter.py#L37-L41); [agent_reach/channels/mcporter.py:104](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/channels/mcporter.py#L104); [agent_reach/skill/SKILL_en.md:65](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/SKILL_en.md#L65)
- Installed third-party tools (mostly unpinned global npm and PyPI packages) run as the user with the full environment and the user's logins. (ASI04; C7). Evidence: [agent_reach/cli.py:1266](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L1266); [agent_reach/cli.py:975](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L975); [agent_reach/utils/process.py:16](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/utils/process.py#L16)

## Criterion details

### C1 Identity & least privilege: 0.25 (high confidence)

Agent Reach has no identity of its own: the host agent runs the upstream tools with the user's existing logins, so the zero-config GitHub channel uses whatever gh login the user has, with full read and write. Credential imports for other platforms are explicit and narrow (only the named cookies for one chosen platform, and install never reads a browser), which is a real narrowing. But every tool Agent Reach launches inherits the full environment, and nothing checks what a request is allowed to do.

- **S L1:** Cookie import is per platform and copies only the named session cookies, but the main channels run on the user's full ambient logins (gh, browser sessions). Evidence: [agent_reach/cookie_extract.py:54](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cookie_extract.py#L54); [docs/README_en.md:115](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/docs/README_en.md#L115) (verified)
  - *To reach the next level:* No read-only or narrowed credential for read-only channels; gh and OpenCLI use the user's full account.
- **C L1:** Cookie-backed channels store only minimal cookies, but every subprocess Agent Reach starts inherits the whole environment and gh/OpenCLI use ambient authority. Evidence: [agent_reach/utils/process.py:16](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/utils/process.py#L16); [agent_reach/probe.py:93](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/probe.py#L93) (verified)
  - *To reach the next level:* Subprocesses and upstream CLIs are not given a scoped identity or a scrubbed environment.
- **D L1:** Install never reads browser credentials and configures none by default, but the agent itself can run the import commands and the GitHub channel uses any existing gh login. Evidence: [agent_reach/cli.py:403](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L403); [docs/README_en.md:115](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/docs/README_en.md#L115) (verified)
  - *To reach the next level:* Widening authority is a command the agent can run without an operator step.
- **B L1:** With the default channels a hijacked agent acts with the user's GitHub login (any repo or org it reaches, including creating issues, PRs and repos); optional channels add social-media sessions. Evidence: [docs/README_en.md:115](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/docs/README_en.md#L115); [agent_reach/skill/references/dev.md:29](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/references/dev.md#L29) (verified)
  - *To reach the next level:* Authority is not confined to one project or to read-only operations.
- **Cap:** none

### C2 Approval gates: 0.17 (high confidence)

Agent Reach ships no approval step of its own. Its installer is check-only unless the --system flag is passed, and it has a dry run, but that flag is passed by the agent itself once it decides the user agreed, so the gate is the model's judgement. The skill says it is for reading only, yet its GitHub reference lists commands that create issues, pull requests, repositories and releases with no risk marking. Anything consequential relies entirely on the host agent's own permission prompts.

- **S L1:** The only consent mechanism is a blanket --system flag for all system installs (with a dry-run preview); the skill's GitHub reference mixes read and write commands with no risk signal. Evidence: [agent_reach/cli.py:270](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L270); [agent_reach/skill/references/dev.md:29](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/references/dev.md#L29); [agent_reach/skill/SKILL_en.md:16-17](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/SKILL_en.md#L16-L17) (verified)
  - *To reach the next level:* No per-action confirmation the model cannot supply, and no risk marking on the write commands the skill lists.
- **C L0:** The most powerful paths the skill directs (arbitrary URL fetch with curl, gh write commands, shell one-liners) pass through no Agent Reach gate. Evidence: [agent_reach/skill/SKILL_en.md:68](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/SKILL_en.md#L68); [agent_reach/skill/references/dev.md:34](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/references/dev.md#L34); [agent_reach/skill/references/web.md:35](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/references/web.md#L35) (verified)
  - *To reach the next level:* Consequential commands the skill directs would need a gate or a read-only restriction.
- **D L1:** Installs are check-only by default, but the agent switches to system changes simply by adding --system. Evidence: [agent_reach/cli.py:270](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L270); [agent_reach/cli.py:91](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L91) (verified)
  - *To reach the next level:* Enabling system changes would need an operator confirmation the model cannot give.
- **B L1:** Wrongly taken actions include public GitHub issues, PRs, repos and releases under the user's account and global package installs, which are not undone by Agent Reach. Evidence: [agent_reach/skill/references/dev.md:34](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/references/dev.md#L34); [agent_reach/skill/references/dev.md:45](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/references/dev.md#L45) (verified)
  - *To reach the next level:* No rollback, preview or quantity bound on the write actions the skill lists.
- **Cap:** C2-SELFAPPROVE: The only consent step, the --system flag, is supplied by the model on its own reading of whether the user approved.

### C3 Tool & action scoping: 0.33 (high confidence)

Agent Reach's own commands validate their inputs reasonably: transcription and its web reader reject non-public literal addresses, config keys are a fixed list, and responses and downloads are size-capped. But the skill routes the real work through general tools, raw curl on any URL, the full gh CLI and API, yt-dlp and shell one-liners, which Agent Reach does not check at all. The URL checks cover literal addresses only and do not follow redirects or DNS. Optional channels are opt-in one by one, but the default set already includes GitHub write and arbitrary fetch.

- **S L2:** Agent Reach's own URL handling rejects non-HTTP schemes, userinfo and literal private addresses, and its config keys are an allowlist, but the checks are literal-only by design. Evidence: [agent_reach/utils/url.py:47](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/utils/url.py#L47); [agent_reach/transcribe.py:215](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/transcribe.py#L215); [agent_reach/transcribe.py:252](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/transcribe.py#L252) (verified)
  - *To reach the next level:* URL checks would need to recheck resolved addresses and redirects.
- **C L1:** Only Agent Reach's own commands validate; the commands the skill tells the agent to run (curl any URL, gh API, yt-dlp) never go through them. Evidence: [agent_reach/skill/SKILL_en.md:68](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/SKILL_en.md#L68); [agent_reach/skill/references/dev.md:48](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/references/dev.md#L48); [agent_reach/skill/SKILL_en.md:74](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/SKILL_en.md#L74) (verified)
  - *To reach the next level:* The skill's main fetch and GitHub paths would need to go through validated, narrow commands.
- **D L2:** Optional channels are installed only when named, but the default zero-config set includes arbitrary URL fetch and the full gh CLI with write commands. Evidence: [agent_reach/cli.py:380](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L380); [agent_reach/skill/references/dev.md:29](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/references/dev.md#L29) (verified)
  - *To reach the next level:* No read-only default tool set; write-capable GitHub commands ship in the default skill.
- **B L0:** A misused tool can fetch any host with curl, call any GitHub API endpoint the user's login allows, and download arbitrary media. Evidence: [agent_reach/skill/SKILL_en.md:68](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/SKILL_en.md#L68); [agent_reach/skill/references/dev.md:48](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/references/dev.md#L48) (verified)
  - *To reach the next level:* Reach is not limited to a project, host allowlist or bounded quantities.
- **Cap:** none

### C4 Code-execution isolation: 0.00 (high confidence)

Agent Reach provides no isolation. The skill has the host agent run shell commands, including Python one-liners and podcast scripts, directly on the user's machine, and the installer runs global npm and pipx installs, whose install scripts execute as the user. Nothing runs in a container or sandbox, so anything that goes wrong has the user's files, network and logins.

- **S L0:** Commands the skill directs and the installer's package installs run as ordinary host processes. Evidence: searched `rg -n -i 'sandbox|seccomp|bwrap|landlock|firejail|nsjail'` in `agent_reach` → 0 hits (No isolation primitive anywhere; every command runs on the host.); [agent_reach/skill/references/web.md:35](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/references/web.md#L35) (verified)
  - *To reach the next level:* No sandbox, container or restricted runtime for any command.
- **C L0:** No execution path is isolated, including package installs that run install scripts. Evidence: [agent_reach/cli.py:1266](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L1266); [agent_reach/skill/references/web.md:35](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/references/web.md#L35) (verified)
  - *To reach the next level:* At least the main command path would need to run in a sandbox.
- **D L0:** There is no isolation to enable. Evidence: searched `rg -n -i 'sandbox|seccomp|bwrap|landlock|firejail|nsjail'` in `agent_reach` → 0 hits (No isolation primitive anywhere; every command runs on the host.) (verified)
  - *To reach the next level:* An isolation mechanism would have to exist and be on by default.
- **B L0:** Commands run with the user's home directory, full environment (including any tokens) and unrestricted network. Evidence: [agent_reach/utils/process.py:16](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/utils/process.py#L16); [agent_reach/cli.py:1266](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L1266) (verified)
  - *To reach the next level:* Commands would need to run without the user's home, credentials and open network.
- **Cap:** none

### C5 Untrusted input blast radius: 0.00 (high confidence)

Agent Reach exists to pour untrusted internet content (web pages, tweets, Reddit, YouTube comments, search results) into an agent that also holds the user's GitHub login and can fetch any URL. Neither the skill nor the install guide marks fetched content as untrusted, records where it came from, or limits what the agent may do after reading it. A prompt injection in any page can therefore steer the agent to leak data through a URL and act on the user's GitHub account without Agent Reach involving a human.

- **S L0:** Nothing limits a hijacked agent; there isn't even prompt-level guidance about untrusted content. Evidence: searched `rg -n -i 'untrusted|prompt injection|quarantin|provenance'` in `agent_reach/skill docs/install.md` → 0 hits (The skill and install guide never mark fetched content as untrusted or restrict actions after reading it.) (verified)
  - *To reach the next level:* Write and egress commands would need to be removed or gated once untrusted content is read.
- **C L0:** No source is distinguished from the user's instructions. Evidence: searched `rg -n -i 'untrusted|prompt injection|quarantin|provenance'` in `agent_reach/skill docs/install.md` → 0 hits (The skill and install guide never mark fetched content as untrusted or restrict actions after reading it.); [agent_reach/skill/SKILL_en.md:68](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/SKILL_en.md#L68) (verified)
  - *To reach the next level:* Fetched content would need provenance or untrusted marking on at least the main sources.
- **D L0:** No control exists to be on by default. Evidence: searched `rg -n -i 'untrusted|prompt injection|quarantin|provenance'` in `agent_reach/skill docs/install.md` → 0 hits (The skill and install guide never mark fetched content as untrusted or restrict actions after reading it.) (verified)
  - *To reach the next level:* A control would need to exist and be on by default.
- **B L0:** In the default setup an injected page can make the agent send data to any URL with curl and create issues, PRs or repos with the user's gh login, unattended by Agent Reach. Evidence: [agent_reach/skill/SKILL_en.md:68](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/SKILL_en.md#L68); [agent_reach/skill/references/dev.md:29](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/references/dev.md#L29); [docs/README_en.md:115](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/docs/README_en.md#L115) (verified)
  - *To reach the next level:* Exfiltration or irreversible actions would need human approval.
- **Cap:** C5-WORSTCASE: Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity: 0.10 (high confidence)

Agent Reach keeps no memory, but two things persist into later sessions without review. Its settings file, including the network proxy that the guides tell agents to export before running tools, can be rewritten by the agent with a plain command. And the search command the skill runs reads MCP server definitions from the current working directory as well as the user's home, so a file in the workspace can change which server starts, with no trust decision. The skill itself is installed only into user-level skill folders, which is good.

- **S L0:** Workspace files can define the MCP servers the skill's search command starts, and the agent can rewrite persistent settings with no prompt. Evidence: [agent_reach/channels/mcporter.py:37-41](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/channels/mcporter.py#L37-L41); [agent_reach/channels/mcporter.py:104](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/channels/mcporter.py#L104); [agent_reach/skill/SKILL_en.md:65](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/SKILL_en.md#L65); [agent_reach/cli.py:1503-1507](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L1503-L1507) (verified)
  - *To reach the next level:* Server definitions should come only from user scope, and settings changes should need confirmation.
- **C L0:** Neither the settings file nor the workspace server definitions are controlled. Evidence: [agent_reach/channels/mcporter.py:104](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/channels/mcporter.py#L104); [agent_reach/cli.py:1503-1507](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L1503-L1507) (verified)
  - *To reach the next level:* At least one persistent path would need validation or a trust decision.
- **D L1:** Settings live in the single user's home directory with owner-only permissions, but nothing stops the agent from changing them. Evidence: [agent_reach/config.py:62](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/config.py#L62); [agent_reach/cli.py:1503-1507](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L1503-L1507) (verified)
  - *To reach the next level:* The model would need to be unable to alter its own settings.
- **B L1:** A poisoned setting or workspace server definition persists across the user's sessions and can steer tool use (which server starts, where traffic is proxied). Evidence: [agent_reach/cli.py:1503-1507](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L1503-L1507); [agent_reach/skill/SKILL_en.md:65](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/SKILL_en.md#L65) (verified)
  - *To reach the next level:* Persisted changes would need to influence only text or gated actions.
- **Cap:** C6-REPOCONFIG: The skill's search command loads server definitions from the workspace's config directory alongside the user's own, with no trust decision.

### C7 Third-party extensions: 0.20 (high confidence)

Agent Reach is an installer for third-party tools, and most of what it installs is unpinned: mcporter, undici and OpenCLI come from global npm installs at whatever version is latest, twitter-cli and bilibili-cli from PyPI likewise, and the setup and update guides the agent follows are read from the main branch. Two Python CLIs are pinned to exact commits, which helps, but nothing is checked against a hash or signature. Installs need the --system flag, which the agent passes itself. Everything installed runs as the user with full access.

- **S L1:** Sources are fixed by Agent Reach but mostly unpinned; rdt-cli and boss-agent-cli are pinned to commits; nothing is hash-verified. Evidence: [agent_reach/cli.py:1266](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L1266); [agent_reach/cli.py:975](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L975); [agent_reach/cli.py:21](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L21); [docs/README_en.md:78](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/docs/README_en.md#L78) (verified)
  - *To reach the next level:* All installed packages and the remote guides would need pinned versions.
- **C L1:** Only the two git-sourced Python CLIs are pinned; npm packages, PyPI CLIs, the remote Exa MCP server and the remote install/update guides are not. Evidence: [agent_reach/cli.py:1096](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L1096); [agent_reach/cli.py:1303](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L1303); [docs/update.md:40](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/docs/update.md#L40) (verified)
  - *To reach the next level:* Pinning would need to cover npm packages and the remote guides too.
- **D L1:** Installs happen behind the generic --system flag, which the agent itself passes after reading the remote guide. Evidence: [agent_reach/cli.py:270](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L270); [docs/README_en.md:78](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/docs/README_en.md#L78) (verified)
  - *To reach the next level:* The exact packages would need to be shown to and confirmed by the user.
- **B L0:** Global npm and pipx tools run as the user with the full environment, and are then run by the agent with the user's logins. Evidence: [agent_reach/cli.py:1266](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L1266); [agent_reach/utils/process.py:16](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/utils/process.py#L16) (verified)
  - *To reach the next level:* Installed tools would need a separate, scrubbed environment at minimum.
- **Cap:** none

### C8 Secrets & sensitive-data protection: 0.50 (high confidence)

Agent Reach handles the secrets it stores carefully: the settings file is written atomically with owner-only permissions and refuses symlinks, secret settings are masked when shown, error and doctor output is scrubbed of credentials in URLs, values can be entered through a hidden prompt, and there is no telemetry. The weak spot is the setup flow: the install guide asks users to paste cookies and API keys to the agent, and the Twitter instructions have the agent put tokens in shell commands, so those secrets pass through the model provider. Tokens are long-lived and stored in plain text.

- **S L2:** Secrets are stored in a plaintext file with owner-only permissions, masked in display, and scrubbed from error and doctor output. Evidence: [agent_reach/config.py:62](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/config.py#L62); [agent_reach/config.py:230](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/config.py#L230); [agent_reach/doctor.py:36](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/doctor.py#L36) (verified)
  - *To reach the next level:* No OS keychain, and secrets are not kept out of model-bound messages.
- **C L2:** Logs (off by default), error messages and doctor output are scrubbed, but the guided setup routes cookies and keys through the agent's chat and shell commands, and subprocesses inherit the full environment. Evidence: [docs/install.md:157](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/docs/install.md#L157); [agent_reach/skill/references/social.md:89](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/references/social.md#L89); [agent_reach/utils/process.py:16](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/utils/process.py#L16) (verified)
  - *To reach the next level:* Model-bound messages and subprocess environments are not protected.
- **D L3:** No telemetry, logging is suppressed unless --verbose, scrubbing is always on, and a hidden prompt is the default way to enter a value. Evidence: searched `rg -n -i 'posthog|sentry|mixpanel|segment\.io|amplitude'` in `agent_reach` → 0 hits (No telemetry or crash-reporting SDK in the package.); [agent_reach/cli.py:63-65](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L63-L65); [agent_reach/cli.py:1419](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L1419); [agent_reach/cli.py:1402](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L1402) (verified)
  - *To reach the next level:* Stored secrets are not encrypted or minimised at rest.
- **B L1:** Stored tokens are long-lived session cookies and API keys (and the ambient gh login) that the agent can read. Evidence: [agent_reach/config.py:230](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/config.py#L230); [docs/README_en.md:115](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/docs/README_en.md#L115) (verified)
  - *To reach the next level:* Credentials would need to be scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability: 0.00 (high confidence)

Agent Reach keeps no record of what the agent did with it. There is no audit log of installs, configuration changes or the upstream commands the skill directs, and its own logging is switched off unless --verbose is passed, and then only goes to the terminal. Reconstructing an incident depends entirely on the host agent's transcript.

- **S L0:** No record of commands, installs or configuration changes is written. Evidence: searched `rg -n -i 'audit|\.jsonl|trajectory'` in `agent_reach` → 0 hits (No audit log, transcript or trajectory writer anywhere in the package.); [agent_reach/cli.py:63-65](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L63-L65) (verified)
  - *To reach the next level:* A structured record of each command and configuration change is missing.
- **C L0:** Nothing is recorded on any path. Evidence: searched `rg -n -i 'audit|\.jsonl|trajectory'` in `agent_reach` → 0 hits (No audit log, transcript or trajectory writer anywhere in the package.) (verified)
  - *To reach the next level:* At least the installer and configure commands would need to be recorded.
- **D L0:** Logging is off by default and goes only to stderr when enabled. Evidence: [agent_reach/cli.py:63-65](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/cli.py#L63-L65) (verified)
  - *To reach the next level:* A durable record would need to be on by default.
- **B L0:** With no record, actions leave no trace in Agent Reach. Evidence: searched `rg -n -i 'audit|\.jsonl|trajectory'` in `agent_reach` → 0 hits (No audit log, transcript or trajectory writer anywhere in the package.) (verified)
  - *To reach the next level:* Records would need to be written per action and surface failures.
- **Cap:** none

### C10 Limits & kill switch: 0.38 (high confidence)

Agent Reach's own commands are well bounded: every subprocess it starts has a timeout, transcription caps file size, audio length and number of chunks, and its web reader caps response size. But those bounds cover only Agent Reach's own commands; the curl, yt-dlp, gh and social-media commands the skill sends the agent to run have no limits from Agent Reach, and the skill encourages fetching from several platforms in parallel. The install guide also suggests a daily scheduled check that keeps running after a session ends.

- **S L2:** Agent Reach enforces timeouts on every subprocess it spawns and size, duration and chunk caps on transcription. Evidence: [agent_reach/transcribe.py:40-43](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/transcribe.py#L40-L43); [agent_reach/transcribe.py:271](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/transcribe.py#L271); [agent_reach/channels/web.py:11](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/channels/web.py#L11) (verified)
  - *To reach the next level:* No rate limits or caps on the upstream commands the skill directs.
- **C L1:** Bounds apply only to Agent Reach's own commands; the skill's direct curl, yt-dlp and platform CLI calls are unbounded by it. Evidence: [agent_reach/skill/SKILL_en.md:68](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/SKILL_en.md#L68); [agent_reach/skill/SKILL_en.md:74](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/SKILL_en.md#L74); [agent_reach/skill/SKILL_en.md:41](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/SKILL_en.md#L41) (verified)
  - *To reach the next level:* The commands the skill directs would need bounds too.
- **D L2:** Limits are hard-coded constants the agent cannot raise through Agent Reach's flags. Evidence: [agent_reach/transcribe.py:40-43](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/transcribe.py#L40-L43); [agent_reach/transcribe.py:271](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/transcribe.py#L271) (verified)
  - *To reach the next level:* The agent can sidestep them by calling the upstream tools directly.
- **B L1:** A runaway agent can keep fetching without ceiling, and the suggested daily scheduled check runs after the session ends. Evidence: [docs/install.md:375](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/docs/install.md#L375); [agent_reach/skill/SKILL_en.md:41](https://github.com/Panniantong/Agent-Reach/blob/a19a171fa980a0785849596492e0af4db800c82f/agent_reach/skill/SKILL_en.md#L41) (verified)
  - *To reach the next level:* No run-level time or cost ceiling, and scheduled work continues after a stop.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Web pages, social posts, comments and search results the skill fetches (agent_reach/skill/SKILL_en.md:68) · [B] sensitive data/systems: The user's gh login and stored tokens in ~/.agent-reach/config.yaml (docs/README_en.md:115) · [C] state change / egress: curl to any URL and gh write commands (agent_reach/skill/SKILL_en.md:68; agent_reach/skill/references/dev.md:29) · Same default session? Yes

## Highest-impact improvements
1. Have the skill run mcporter with an explicit user-level config (or from a fixed directory) so workspace files can't define the servers it starts, and require confirmation for settings changes. (C6 S L0→L2, +0.150 before caps; Playbook 2)
2. Pin every installed package (mcporter, undici, OpenCLI, twitter-cli, bilibili-cli) to exact versions with hashes, and point the install and update guides at a release tag instead of main. (C7 S L1→L3, +0.150 before caps; Playbook 3)
3. Make --system require an interactive confirmation listing the exact packages, and drop the GitHub write commands from the read-only skill or mark them as needing user approval. (C2 S L1→L2, +0.075 before caps; Playbook 5)
4. Route web reads through Agent Reach's own validated reader (public-address check, size cap) instead of raw curl, and recheck addresses after DNS and redirects. (C3 C L1→L2, +0.075 before caps; Playbook 3)
5. Wrap fetched content with its source and an untrusted marker, and tell the host to stop write and egress commands once untrusted content is read. (C5 S L0→L1, +0.075 before caps; Playbook 1)

## Re-audit log
- C3 B: L1 → L0. Re-read the L0 anchor: the skill's default tools include curl to any host and the generic gh API, which are general-purpose tools with no host or endpoint bound; the minor limits credited at L1 apply only to Agent Reach's own commands, which the skill does not route web reads through.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Agent Reach does not own the agent loop: approval, context handling and logging belong to the host agent (Claude Code, OpenClaw, Cursor and others), which were not scored and are not credited.
- Behaviour of upstream tools (mcporter, OpenCLI, twitter-cli, gh, yt-dlp) is taken from Agent Reach's own code comments and docs, not verified in their source.
- Optional channels (Twitter, Reddit, XiaoHongShu, Facebook, Instagram, Boss, LinkedIn, Xiaoyuzhou) are footnoted, not scored; they add browser sessions and cookies to the agent's authority.
- The MCP server exposes only a read-only status tool and was not scored separately.
- No release tag at the pinned commit; pyproject.toml declares version 1.5.0.
