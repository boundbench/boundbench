# Defense-in-Depth Score: AI Town

**Repo:** https://github.com/a16z-infra/ai-town · **Commit:** `8e05997f2409275669c8344b84a51692e83f3f33` · **Reviewed:** 2026-10-04
**What it is:** A deployable starter kit for a virtual town where LLM-driven characters live, chat and socialise, built on Convex, with human players able to join.
**Category:** Agent Frameworks
**Scored configuration:** README standard setup: Convex backend with the default local Ollama provider, optional Clerk auth not added, public web frontend where anyone can join as a human player.
**Agent surface (default):** code execution no · filesystem write no · network egress no · external credentials opt-in · persistent memory yes · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 6.2 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L1 | L1 | L2 | 0.38 | — | **0.38** | High |
| C2 | Approval gates | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C3 | Tool & action scoping | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L3 | 0.40 | — | **0.40** | High |
| C6 | Memory, context & configuration integrity | L2 | L1 | L0 | L2 | 0.33 | — | **0.33** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L2 | L0 | L2 | 0.33 | — | **0.33** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L3 | 0.55 | — | **0.55** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | G2 | **0.25** | High |

Controls where a risk surface exists: 2.23 / 6.0 (37%); 4 criteria scored SA (surface absent).

AI Town's agents have a tiny action surface: the model never calls tools or runs code, and everything it produces is in-game chat text or a memory summary, which is why four criteria score as structurally absent. The real risks sit around the agents rather than in them: authentication is not configured by default, so anonymous visitors share one player identity and can plant persistent memories that every other user's conversations will draw on, and the public backend functions do not enforce access control on every path. Every LLM prompt and response is also logged in full, and there is no overall spend cap.

## Critical gaps
- None: no criterion is capped and no blast radius is at the worst level.

## Criterion details

### C1 Identity & least privilege — 0.38 (high)

The AI agents themselves hold almost no authority: their output is plain text that the backend stores as a chat message or a memory, and the only credential is the operator's LLM API key, used solely as an HTTP header to the configured model endpoint (none at all with the default local Ollama). The gap is the human side of the trust boundary: authentication is not configured by default, every human player is the same identity 'Me', and the public backend functions do not enforce authorization on every path. The damage stays inside this one game database.

- **S L2:** Agent code paths are internal Convex functions with narrow arguments and one deployment-wide LLM key; the model cannot reach the database or any other system except through fixed message/memory writes. — [convex/util/llm.ts:112-117](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/util/llm.ts#L112-L117); [convex/aiTown/agent.ts:307-325](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/aiTown/agent.ts#L307-L325) (verified)
  - *To reach the next level:* No per-capability or per-requester scoping: one static key and full backend DB authority serve every agent and every visitor.
- **C L1:** Authorization on the public mutation paths is not locked down. (verified)
  - *To reach the next level:* Requests on the public paths are not tied to an authenticated principal.
- **D L1:** Default install has no authentication: the identity check is commented out and every human joins as the shared DEFAULT_NAME; adding auth (Clerk) is an optional manual step. — [convex/world.ts:111-138](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/world.ts#L111-L138); searched `rg -n -S 'getUserIdentity'` in `convex` → 4 hits (All 4 hits are commented-out identity checks in convex/world.ts.) (verified)
  - *To reach the next level:* No authenticated principal by default; least privilege for visitors requires manual hardening.
- **B L2:** Blast radius is confined to one system, the game's Convex database (messages, world state, agents), plus LLM spend; the API key is never returned to clients. (verified)
  - *To reach the next level:* Writes are not limited to the caller's own scope.
- **Cap:** none

### C2 Approval gates — 1.00 (high)

The agents have no consequential actions to approve. The model never calls tools: everything it produces is chat text shown inside the game, a memory summary, or a number, and all movement and invitations are chosen by deterministic game code. Nothing it writes leaves the app, deletes data, or spends money directly. Memory persistence is scored under C6 and the effect of hostile chat under C5.

- **Structural absence:** searched `rg -n -S 'tool_choice|tools:|function_call|tool_calls'` in `convex src/components src/hooks src/App.tsx` → 3 hits (All 3 hits are unused optional fields in the copied OpenAI request/response type definitions in convex/util/llm.ts (one commented out); no request ever sets tools, so the model has no tool calling.); searched `rg -n -S 'child_process|eval\(|new Function|execSync|spawn\('` in `convex src/components src/hooks src/App.tsx` → 0 hits (No process spawning, eval, or dynamic code construction in backend or app frontend (map editor under src/editor excluded: standalone dev tool, not reachable by the agent).)

### C3 Tool & action scoping — 1.00 (high)

There are no model-invoked tools to scope. Agent actions (wander, pick an activity, invite a nearby player) are chosen by game code from fixed lists and map coordinates, not by the model, and model text only becomes a chat message or memory. The human-facing public API is covered under C1.

- **Structural absence:** searched `rg -n -S 'tool_choice|tools:|function_call|tool_calls'` in `convex src/components src/hooks src/App.tsx` → 3 hits (All 3 hits are unused optional fields in the copied OpenAI request/response type definitions in convex/util/llm.ts (one commented out); no request ever sets tools, so the model has no tool calling.); [convex/aiTown/agentOperations.ts:127-129](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/aiTown/agentOperations.ts#L127-L129)

### C4 Code-execution isolation — 1.00 (high)

No model-generated or user-supplied text is ever run as code. The backend has no shell, eval, or process spawning; model output is stored as strings, and the one structured output (reflection JSON) is only parsed with JSON.parse and used as array indices.

- **Structural absence:** searched `rg -n -S 'child_process|eval\(|new Function|execSync|spawn\('` in `convex src/components src/hooks src/App.tsx` → 0 hits (No process spawning, eval, or dynamic code construction in backend or app frontend (map editor under src/editor excluded: standalone dev tool, not reachable by the agent).); [convex/agent/memory.ts:371-374](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/agent/memory.ts#L371-L374)

### C5 Untrusted input blast radius — 0.40 (high)

Anyone who opens the app can join anonymously and type messages that go straight into the agents' prompts, with no length limit and no screening (a moderation helper exists but is never called). Retrieved memories are wrapped as JSON marked untrusted with a prompt telling the model not to follow them, which is only a soft defence, and live chat messages get no such wrapping. What limits the damage is the small surface: a hijacked agent can only produce a short chat line inside the game and a poisoned memory, has no tools or outbound channel, and the data it can see (other in-game conversations) is already publicly viewable.

- **S L1:** Only delimiting/spotlighting: memories are serialized as JSON with trust:'untrusted' plus a prompt instruction; nothing in code acts on the tag. — [convex/agent/conversation.ts:221-245](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/agent/conversation.ts#L221-L245) (verified)
  - *To reach the next level:* No code-enforced restriction once untrusted content is read; the defence is prompt-level.
- **C L1:** The untrusted wrapper covers retrieved memories only; live human chat enters as plain user-role text, and the moderation helper is unused. — [convex/agent/conversation.ts:256-262](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/agent/conversation.ts#L256-L262); searched `rg -n 'fetchModeration'` in `convex` → 1 hits (Only the definition; never called.) (verified)
  - *To reach the next level:* Live chat messages, summarisation and reflection prompts are not covered by any provenance handling.
- **D L2:** The wrapper is hard-coded and cannot be configured away by content, but a prompt-level mechanism earns at most one level above its strength. — [convex/agent/conversation.ts:49-61](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/agent/conversation.ts#L49-L61) (verified)
  - *To reach the next level:* Mechanism strength limits this; a code-enforced control would be needed for higher credit.
- **B L3:** A hijacked agent can only emit in-game chat (max 300 tokens) and write its own memory; it has no tools, no egress it controls, and only low-sensitivity data that is already publicly queryable. — [convex/agent/conversation.ts:131-135](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/agent/conversation.ts#L131-L135); [convex/messages.ts:6-15](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/messages.ts#L6-L15) (verified)
  - *To reach the next level:* Sessions that read untrusted chat still change persistent state (memory writes), so not text-only.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.33 (high)

After every conversation the agent's LLM summarises the chat, including whatever an anonymous visitor typed, into a persistent memory that is later retrieved into prompts with every other player. Memories are filtered per agent, but because all humans share one identity they are effectively shared across all users; nothing validates or reviews what gets written. Retrieved memories are presented to the model as untrusted JSON in conversation prompts, but the reflection and importance-scoring prompts insert them raw. The effect is limited to what agents say, and memories are vacuumed after two weeks.

- **S L2:** Memory entries are presented to the conversation model as data tagged untrusted, but writes are unvalidated LLM summaries of untrusted chat. — [convex/agent/memory.ts:55-71](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/agent/memory.ts#L55-L71); [convex/agent/conversation.ts:231-245](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/agent/conversation.ts#L231-L245) (verified)
  - *To reach the next level:* Memory writes are not gated, validated, or source-restricted, and have no review step.
- **C L1:** The untrusted tag covers conversation prompts only; reflection and importance prompts embed memory text raw and reflections are written back as new memories. — [convex/agent/memory.ts:350-353](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/agent/memory.ts#L350-L353); [convex/agent/memory.ts:252](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/agent/memory.ts#L252) (verified)
  - *To reach the next level:* Reflection and importance paths are not covered by the provenance handling.
- **D L0:** Memory is namespaced per agent in the vector query, but with auth disabled every human is 'Me', so one visitor's poisoned memory reaches every other user's conversations. — [convex/agent/memory.ts:164-167](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/agent/memory.ts#L164-L167); [convex/world.ts:122](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/world.ts#L122) (verified)
  - *To reach the next level:* No per-user isolation of memory by default.
- **B L2:** Poisoned memories persist across sessions and users but can only influence chat text (agents have no tools); they are inspectable in the Convex dashboard and auto-deleted after 14 days. — [convex/constants.ts:62](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/constants.ts#L62); [convex/crons.ts:30-37](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/crons.ts#L30-L37) (verified)
  - *To reach the next level:* Poisoned memories are not session-scoped or purged on demand; they reach all users for up to two weeks.
- **Cap:** none
- **Notes:** D is L0 on the isolation anchor (memory shared across all users), not because the mechanism is opt-in; G1 does not apply.

### C7 Third-party extensions — 1.00 (high)

The app loads no plugins, MCP servers, or downloaded tools at runtime. One related behaviour: with the default Ollama provider, a missing model is automatically pulled from the Ollama registry by the name the operator configured; these are model weights run by the separate Ollama process, not code loaded by AI Town.

- **Structural absence:** searched `rg -n -S 'mcp|plugin|require\(|import\('` in `convex src/components src/hooks src/App.tsx` → 3 hits (Hits: a Pixi viewport plugin comment, a dynamic import of Node's built-in 'crypto' in embeddingsCache.ts, and a commented require in a vendored compression util. None loads third-party code chosen at runtime.)
- **Notes:** Ollama auto-pull on 404 (convex/util/llm.ts:162-163, 190-200) fetches an unpinned operator-named model; not counted as third-party code.

### C8 Secrets & sensitive-data protection — 0.33 (high)

API keys come from Convex environment variables and are only ever placed in the Authorization header to the model provider, never in prompts or responses. However, every chat completion request and response, including all player chat and memories, is written to the Convex logs with console.log by default, and failed provider responses are logged verbatim. There is no redaction or telemetry; with the default Ollama setup there is no key at all, while OpenAI or Together keys are long-lived.

- **S L1:** Secrets are read from environment variables and kept out of model context; there is no masking or redaction of logged payloads. — [convex/util/llm.ts:46-59](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/util/llm.ts#L46-L59); [convex/util/llm.ts:144](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/util/llm.ts#L144) (verified)
  - *To reach the next level:* No redaction layer or log filter on the payload logging paths.
- **C L2:** By design the key reaches only the outbound header (not model messages, client responses, or subprocesses), but logs carry full prompt and completion content. — [convex/util/llm.ts:112-117](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/util/llm.ts#L112-L117); [convex/util/llm.ts:178](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/util/llm.ts#L178) (verified)
  - *To reach the next level:* Logs and error paths carry unredacted payloads.
- **D L0:** Verbose payload logging of every LLM request and response is on by default with no switch. — [convex/util/llm.ts:140-145](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/util/llm.ts#L140-L145) (verified)
  - *To reach the next level:* Payload logging is unconditional; there is no default-off or redacted mode.
- **B L2:** Default Ollama holds no key; configured provider keys (OpenAI, Together, Replicate) are single-purpose but long-lived, and are not reachable by the model. — [convex/util/llm.ts:102-109](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/util/llm.ts#L102-L109); [convex/music.ts:7-11](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/music.ts#L7-L11) (verified)
  - *To reach the next level:* Keys are long-lived with no rotation or short-lived issuance.
- **Cap:** none
- **Notes:** D is L0 because payload logging is on by default, not because a mechanism is opt-in; G1 does not apply.

### C9 Audit & traceability — 0.55 (high)

Every game action, from humans and agents alike, is written as a numbered input row (name, arguments, receive time, result) before the engine processes it, and every chat message is stored, so the simulation can be reconstructed. The record has no real actor attribution, though: all humans are 'Me', and attribution is not tamper-resistant. LLM calls appear only in unstructured logs, and the input history is deleted after two weeks by a default cron job.

- **S L2:** Structured, numbered record of every engine input with arguments, timestamp and return value. — [convex/engine/abstractGame.ts:133-154](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/engine/abstractGame.ts#L133-L154); [convex/engine/schema.ts:4-33](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/engine/schema.ts#L4-L33) (verified)
  - *To reach the next level:* No actor attribution separating agent, human principal, or caller.
- **C L2:** All human and agent game actions pass through the inputs table and messages table; LLM calls and memory writes are only in console logs or the memories table itself. — [convex/aiTown/agent.ts:318-332](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/aiTown/agent.ts#L318-L332); [convex/util/llm.ts:144](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/util/llm.ts#L144) (verified)
  - *To reach the next level:* LLM calls and memory writes are not part of the structured record.
- **D L2:** On by default and outside model control, but the same backend deletes inputs after 14 days via a default cron, and record attribution is not tamper-resistant. — [convex/crons.ts:18-30](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/crons.ts#L18-L30); [convex/constants.ts:62](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/constants.ts#L62) (verified)
  - *To reach the next level:* Records are auto-deleted by default and their attribution is weak.
- **B L3:** An input is durably stored in a transaction before the engine acts on it, so game actions have a replayable trail. — [convex/engine/abstractGame.ts:146-152](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/engine/abstractGame.ts#L146-L152) (verified)
  - *To reach the next level:* Durability ends with the 14-day vacuum, and LLM calls are only best-effort console logs.
- **Cap:** none

### C10 Limits & kill switch — 0.25 (high)

The simulation has sensible local limits: chat replies are capped at 300 tokens, conversations end after 8 messages or 10 minutes, agent operations time out after 2 minutes, LLM retries are bounded, and a world stops itself after 5 minutes with no viewers. But there is no overall token or cost budget, the reflection call has no token cap, a world stays alive while any visitor has it open, and the kill switch and agent-creation limits are not enforced on every path.

- **S L2:** Per-call token caps, per-conversation message and duration caps, and operation timeouts are enforced in code; halt is cooperative via the engine running flag. — [convex/constants.ts:41-45](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/constants.ts#L41-L45); [convex/aiTown/agent.ts:57-60](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/aiTown/agent.ts#L57-L60) (verified)
  - *To reach the next level:* No wall-clock or token/cost budget for the world as a whole, and no rate limit on agent creation.
- **C L2:** Limits apply to the engine loop and agent operations, but the reflection completion has no max_tokens and in-flight LLM actions continue after a stop. — [convex/agent/memory.ts:362-369](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/agent/memory.ts#L362-L369); [convex/aiTown/main.ts:77-87](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/aiTown/main.ts#L77-L87) (verified)
  - *To reach the next level:* Not all LLM calls are bounded, and stop does not cancel scheduled or in-flight actions.
- **D L1:** Defaults are reasonable, but the operator's stop and agent-count limits are not enforced on every path. (verified)
  - *To reach the next level:* Operator limits are not robust against untrusted clients.
- **B L1:** There is no spend ceiling: as long as any visitor keeps the page open, agents keep making LLM calls indefinitely, and stop leaves in-flight actions running. — [convex/constants.ts:4](https://github.com/a16z-infra/ai-town/blob/8e05997f2409275669c8344b84a51692e83f3f33/convex/constants.ts#L4) (verified)
  - *To reach the next level:* No per-world spend ceiling and no cancellation of in-flight work on stop.
- **Cap:** G2 — Kill-switch enforcement does not cover every path.

## Rule-of-Two check
[A] untrusted input: Anonymous human chat enters prompts (convex/agent/conversation.ts:256-262). · [B] sensitive data/systems: Low sensitivity: other players' conversation memories retrieved per agent (convex/agent/memory.ts:158-173); the same conversations are publicly queryable. · [C] state change / egress: Persistent memory writes (convex/agent/memory.ts:273-288) and in-game chat (convex/aiTown/agent.ts:307-325); no outbound channel the model controls. · Same default session? Yes

## Highest-impact improvements
1. Harden the kill switch so only the operator controls world lifecycle. — C10 D L1→L2, +0.050 before caps (Playbook 3 step 3)
2. Enable authentication by default and authorize every public mutation against the caller's own player. — C1 C L1→L3, +0.150 before caps (Playbook 4)
3. Remove or guard the unconditional console.log of every LLM request and response behind a debug flag. — C8 D L0→L2, +0.100 before caps (Playbook 4)
4. Apply the untrusted-memory envelope to the reflection and importance prompts too, so all memory reads are tagged. — C6 C L1→L2, +0.075 before caps (Playbook 2)
5. Add a per-world LLM call/token budget and a max_tokens on the reflection completion. — C10 B L1→L2, +0.050 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Convex platform behaviour (action timeouts, scheduler semantics, log retention) is assumed from its documented model, not verified in Convex's own code.
- The standalone map/sprite editor under src/editor and the Fly/Vercel deployment configs were not examined in depth.
- No text aimed at AI reviewers was found in the repository.
