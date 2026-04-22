# SillyTavern Extension — Secret Plot Driver

A faithful port of the **`secret-plot-driver`** agent from the
**[Marinara Engine](https://github.com/Pasta-Devs/Marinara-Engine)** by
[Pasta-Devs](https://github.com/Pasta-Devs), adapted into a stand-alone
SillyTavern extension.

All credit for the agent's design — the prompt, the pipeline, the
persistence rules, the pacing/staleness logic — belongs to the
Marinara Engine authors. This extension only re-wires that same
pipeline into SillyTavern's extension API so users who don't run
Marinara's full stack can still get the Secret Plot Driver behavior.

A hidden *Narrative Architect* that:

1. **Before** every main generation, reads the recent chat and its own
   previous state (via `generateRaw` — the agent task is sent as a
   `role: user` message, not a system message, so models like Claude
   cleanly break out of the roleplay to respond with JSON), then
   produces a JSON blob describing:
   - a long-term **overarching arc** (mystery, resolution conditions,
     protagonist journey, whether it's completed)
   - a single active **scene direction** — a short, one-sentence
     directorial nudge (mood/tone/trajectory, never beat-by-beat
     choreography)
   - current **pacing mode** (`slow` / `exploration` / `building` /
     `climactic` / `cooldown`)
   - **staleness detection** (forces change when the scene has lost
     momentum)
2. Persists that state to the chat's `chat_metadata` so it survives
   page reloads and chat reopens.
3. Injects the arc + the active direction into the main prompt via
   SillyTavern's extension-prompt system, so the main character-gen
   model writes toward them without the user ever seeing the agent's
   reasoning.
4. **Does not** re-run on swipes/regens — the stored plot state stays
   consistent across swipes. It only runs on real user turns.

## How it works (the pipeline)

```
user hits send
     │
     ├── GENERATION_AFTER_COMMANDS fires (awaited)
     │        │
     │        └── Secret Plot Driver hook (this extension, makeFirst)
     │                 │
     │                 ├── capture pending user text from #send_textarea
     │                 │     (it hasn't been pushed into chat[] yet at this point)
     │                 │
     │                 ├── build message array:
     │                 │     system  — "You are a backend analysis agent, NOT a
     │                 │               roleplay character…"
     │                 │     user    — <recent_roleplay_transcript> header
     │                 │     user/assistant — last N chat messages (full text)
     │                 │     user    — the pending message (just captured)
     │                 │     user    — <secret_plot_state> + <agent_instructions>
     │                 │               + "Respond with JSON ONLY"
     │                 │
     │                 ├── generateRaw(prompt: messages)
     │                 │     → standalone LLM call with our own message array
     │                 │       (no persona/char card/WI — just chat history +
     │                 │       the agent task, sent as role: user)
     │                 │
     │                 ├── parse JSON { overarchingArc, sceneDirections, pacing,
     │                 │                staleDetected }
     │                 ├── persist to chat_metadata.secret_plot_driver
     │                 │     · overarchingArc (kept until completed)
     │                 │     · sceneDirections (unfulfilled only)
     │                 │     · recentlyFulfilled (rolling last 10)
     │                 │     · pacing / staleDetected
     │                 │
     │                 └── setExtensionPrompt:
     │                       · ARC   → IN_PROMPT (after story string)
     │                       · DIR   → IN_CHAT at depth 0
     │                       (role for each is configurable: system/user/assistant)
     │
     ├── other extensions (Stepped Thinking, etc.) run here
     │       → their quiet generations automatically see the arc/directions
     │         because setExtensionPrompt is global
     │
     └── main generation runs with arc + directions baked into the prompt
```

The agent prompt (`DEFAULT_PROMPT_TEMPLATE` in `index.js`) defines the
full "Narrative Architect" role. State persistence rules: arc survives
until completed, scene directions are kept only while unfulfilled,
fulfilled directions roll into a 10-entry "recentlyFulfilled" window
so the agent doesn't re-issue them.

## Installation

### Option A — via SillyTavern's Extensions menu

1. Extensions → Install Extension
2. Paste the repository URL (or a local path) and install.

### Option B — manual

Copy this folder to:

```
<SillyTavern>/data/<user>/extensions/st-secret-plot-driver/
```

Restart SillyTavern, then enable it in **Extensions**.

## Usage

1. Open a chat.
2. Extensions panel → **Secret Plot Driver** drawer → tick *Enable*.
3. Send a message. The first turn generates the initial arc + direction
   (you'll briefly see a "Composing plot threads..." toast). Subsequent
   turns update or replace them.
4. The drawer always shows the **current arc** and the **active scene
   direction** so you can see what the agent is steering toward.
5. *Reset plot state (this chat)* wipes the stored arc/directions if you
   want a fresh canvas.
6. *Run now* triggers the agent immediately, without sending a message —
   useful for kicking off a plot at the start of a chat.

### Pause vs Disable

- **Disable** — agent never runs, *and* nothing is injected.
- **Pause** — agent stops running (saves tokens), but the last computed
  arc/direction **stays injected** into every generation. Useful when
  you're happy with the current plot and just want it to hold.

## Works alongside Stepped Thinking

Secret Plot Driver registers on `GENERATION_AFTER_COMMANDS` with
`makeFirst`, so it runs **before** Stepped Thinking. Because injection
happens via `setExtensionPrompt` (which is global), Stepped Thinking's
`generateQuietPrompt` calls for character thoughts will include the
fresh secret-plot arc and scene direction automatically — characters
will think *about* what the Director just decreed for this turn.

## Advanced settings

All exposed in the drawer's *Advanced Settings* section:

| Setting | Default | Notes |
|---|---|---|
| Run every N user messages | `1` | `1` = every turn. Higher values save tokens on long chats. |
| Chat context size | `10` messages | How many recent chat messages the agent sees as reference material. |
| Per-message char cap | `0` (no truncation) | Max characters per chat message sent to the agent. `0` sends messages in full — recommended so the agent has complete context. Only set above 0 if you're hitting context-length errors. |
| Agent response length | `2048` tokens | Max tokens for the agent's JSON output. |
| Arc injection role / position / depth | `System` / `After Main Prompt` / `0` | Where the arc block goes in the **main** generation. Role is honored verbatim in the outgoing request — switch to `User` or `Assistant` if you want the arc sent as that role instead of `system`. |
| Direction injection role / position / depth | `System` / `In-Chat` / `0` | Where the scene direction goes. `In-Chat @ depth 0` = right above the latest user message. Role is honored the same way as the arc. |
| Arc injection template | `<overarching_arc>{{arc}}</overarching_arc>` | `{{arc}}` is replaced with the current arc text. |
| Direction injection template | `<scene_directions>{{directions}}</scene_directions>` | `{{directions}}` is replaced with a bullet list of active directions. |
| Agent prompt template | full Narrative Architect prompt | Editable. *Restore default* brings back the original prompt. |

## Credits

**All agent design credit belongs to [Pasta-Devs](https://github.com/Pasta-Devs)
and the [Marinara Engine](https://github.com/Pasta-Devs/Marinara-Engine) project.**

Specifically, the prompt, pipeline, persistence rules, and
pacing/staleness logic are taken from these files in Marinara Engine:

- [`packages/shared/src/constants/agent-prompts.ts`](https://github.com/Pasta-Devs/Marinara-Engine/blob/main/packages/shared/src/constants/agent-prompts.ts)
  — the verbatim "Narrative Architect" prompt and JSON schema.
- [`packages/shared/src/types/agent.ts`](https://github.com/Pasta-Devs/Marinara-Engine/blob/main/packages/shared/src/types/agent.ts)
  — the `secret-plot-driver` agent registration as a `pre_generation`
  phase agent producing `secret_plot` results.
- [`packages/server/src/services/agents/agent-executor.ts`](https://github.com/Pasta-Devs/Marinara-Engine/blob/main/packages/server/src/services/agents/agent-executor.ts)
  — the generic agent runner (state serialization, JSON repair logic,
  chat-history assembly).
- [`packages/server/src/routes/generate.routes.ts`](https://github.com/Pasta-Devs/Marinara-Engine/blob/main/packages/server/src/routes/generate.routes.ts)
  — the orchestration: when to run the agent, how to persist its
  output, how the arc gets re-injected into `<lore>` and the scene
  directions into `<context>` for the main generation.

This extension is a port of that pipeline into SillyTavern's extension
API — please support the original project if you find this useful.

## License

MIT
