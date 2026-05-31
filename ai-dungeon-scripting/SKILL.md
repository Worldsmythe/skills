---
name: ai-dungeon-scripting
description: >
  Write and debug AI Dungeon JavaScript scenario scripts: the onInput, onModelContext, and
  onOutput hooks, the modifier pattern, persistent state and memory overrides
  (Plot Essentials / Author's Note / frontMemory), story-card APIs, return-value semantics,
  and the sandbox (including the cacheEfficient / Optimized-Context model limitation). Use
  when authoring, combining, or troubleshooting AI Dungeon scripts. For which community
  scripts exist and how to install them, see the ai-dungeon-scenario-design skill's catalog;
  for the aid CLI that uploads scripts, see the ai-dungeon skill.
---

# AI Dungeon Scripting

The AI Dungeon JavaScript scripting system. Scripts attach to **Scenarios** (not Adventures)
and run in an isolated sandbox. This skill covers how to write them; for the catalog of
community scripts (clone links, per-script compatibility, install/combine steps) see the
`ai-dungeon-scenario-design` skill, and for the `aid scripts` command that uploads them see
the `ai-dungeon` skill.

Full function signatures (Story Card APIs, `info`, `history`, TypeScript declarations) live
in `references/scripting-api.md`.

## Script Slots and Hooks

Four editor slots, visible under a scenario's Details tab:

| Slot        | Hook              | Receives                        | Can stop AI call? |
|-------------|-------------------|---------------------------------|-------------------|
| **Library** | (prepended to all)| Shared utility code             | N/A               |
| **Input**   | `onInput`         | Player's raw input text         | Yes (`stop: true`)|
| **Context** | `onModelContext`  | Fully assembled context string  | No (errors)       |
| **Output**  | `onOutput`        | AI's generated response text    | No (errors)       |

Scripts work on **Simple Start** and **Character Creator** scenarios. Multiple Choice
scenario nodes don't support scripts, but their child leaf options can have scripts if
those leaves are Simple Start or Character Creator type.

Only the scenario creator sees scripts, logs, and the Inspect modal. Scripts have a
`scriptsEnabled` toggle per branch.

---

## The Modifier Pattern

Every non-Library script must end with `modifier(text)` and the modifier function must
return `{ text: string, stop?: boolean }`:

```javascript
const modifier = (text) => {
  let modifiedText = text

  // Your logic here

  return { text: modifiedText }
}

// This line is required — don't modify it
modifier(text)
```

---

## Globals

Available in all hooks:

| Global       | Type          | Description                                    |
|--------------|---------------|------------------------------------------------|
| `state`      | `State`       | Persistent per-adventure state object          |
| `history`    | `History[]`   | Recent actions array                           |
| `info`       | `Info`        | Adventure metadata                             |
| `storyCards` | `StoryCard[]` | Mutable array of story cards                   |
| `text`       | `string`      | Current text (also passed as modifier param)   |
| `log(msg)`   | `function`    | Log to script editor console                   |

Plus imperative Story Card functions: `addStoryCard`, `updateStoryCard`, `removeStoryCard`
(full signatures in `references/scripting-api.md`, along with the `info` and `history` field
lists).

---

## State and Memory

`state` is a free-form persistent object scoped per-adventure. It survives across turns
but each script execution is isolated — use `state` to pass data between turns.

### Reserved Fields

```javascript
// Override Plot Essentials (empty string falls back to UI value)
state.memory.context = "You are a warrior named Kael."

// Override Author's Note (empty string falls back to UI value)
state.memory.authorsNote = "Dark fantasy, terse prose, second person."

// Append to the very end of context, after the last player input
// Hidden from player, visible to AI.
state.memory.frontMemory = "Kael feels a chill run down his spine as"

// Show an info message to the player (not part of the prompt)
state.message = "Quest updated: Find the silver key."

// Multiplayer: selective visibility
state.message = { text: "You hear a whisper.", visibleTo: ["Kael"] }
```

**Critical timing**: changes to `state.memory.*` made in `onOutput` only take effect on
the **next turn**.

### Custom State
```javascript
state.inventory = state.inventory || []
state.inventory.push("silver key")
state.questLog = { findKey: true, defeatBoss: false }
state.turnsSinceCombat = (state.turnsSinceCombat || 0) + 1
```

---

## Return Value Semantics

### onInput
| Return                        | Behavior                                    |
|-------------------------------|---------------------------------------------|
| `{ text: "modified" }`       | Replaces player input, AI is called         |
| `{ text: "" }`               | Error shown to player                       |
| `{ text: null, stop: true }` | Suppresses AI call (for meta-commands)      |

### onModelContext
| Return                  | Behavior                                          |
|-------------------------|---------------------------------------------------|
| `{ text: "modified" }` | Replaces entire context sent to AI                |
| `{ text: "" }`         | Silently rebuilds context as if script didn't run |

### onOutput
| Return                  | Behavior                                    |
|-------------------------|---------------------------------------------|
| `{ text: "modified" }` | Replaces AI output shown to player          |
| `{ text: "" }`         | Error shown to player                       |

---

## Sandbox Constraints

- **Memory limit**: 16 MB
- **Execution timeout**: 2 seconds per hook
- **Console logs**: persist 15 minutes, visible only to scenario creator
- **No network access**: no fetch, no XMLHttpRequest, no imports
- **No DOM**: pure JavaScript computation only
- **Optimized Context / `cacheEfficient` mode** (DeepSeek V4 Flash, Equinox, Gemma 31B,
  DeepSeek V4 Pro, GLM 5.1) caches the prompt. The `onModelContext` modifier still **runs**,
  so *reading* the context (to detect mentions, update `state`, score sentiment) works — but
  any **change it writes to the context text is discarded**. Practically: a feature that
  injects instructions/thoughts/cards/stats by returning modified context text is
  **non-functional** on these models. Payloads delivered through `state.memory.frontMemory`
  or `state.memory.authorsNote` (set from any hook, not the returned context text) survive,
  as do `onInput`/`onOutput` features. The `ai-dungeon-scenario-design` skill's script
  catalog flags this per script.

---

## Practical Patterns

### Command Parser (onInput)

```javascript
const modifier = (text) => {
  const match = text.match(/^:(\w+)\s*(.*)/)
  if (match) {
    const [, cmd, args] = match
    if (cmd === "status") {
      state.message = `Inventory: ${(state.inventory || []).join(", ") || "nothing"}`
      return { text: null, stop: true }
    }
  }
  return { text }
}
modifier(text)
```

### Dynamic Author's Note (onOutput)

Steer tone from state by writing the reserved memory field, not by splicing the context
text (that splice is discarded on `cacheEfficient` models; this isn't):

```javascript
const modifier = (text) => {
  state.memory.authorsNote = state.inCombat
    ? "Tense, fast-paced, second person, present tense."
    : "Atmospheric, contemplative, second person, present tense."
  return { text }
}
modifier(text)
```

### Quest Tracker (onInput)

```javascript
const modifier = (text) => {
  state.quests = state.quests || { findKey: false, defeatDragon: false }
  if (text.toLowerCase().includes("silver key")) {
    state.quests.findKey = true
    state.message = "Quest complete: Found the silver key!"
  }
  const active = Object.entries(state.quests)
    .filter(([, done]) => !done).map(([q]) => q).join(", ")
  if (active) state.memory.frontMemory = `Active quests: ${active}.`
  return { text }
}
modifier(text)
```

---

## Installing, Combining, and the Catalog

To install a community script, **clone its canonical repo and upload it as-is** with the
`aid scripts` command (in the `ai-dungeon` skill) — never recreate a script by hand or by
reading-then-rewriting the live scenario's scripts. To combine scripts, fold each one's
per-slot hook call into one ordered modifier per slot, with every library concatenated in
the Library slot. The `ai-dungeon-scenario-design` skill's **script catalog** lists what
exists, each one's chain call, the compatibility matrix, and which features go inert on
`cacheEfficient` models — pick and install there; this skill is how the hooks behave.
