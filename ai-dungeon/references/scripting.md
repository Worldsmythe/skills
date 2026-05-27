# AI Dungeon Scripting Reference

Complete reference for the AI Dungeon JavaScript scripting system. Scripts attach to
Scenarios (not Adventures) and run in an isolated sandbox.

## Table of Contents
1. [Script Slots and Hooks](#script-slots-and-hooks)
2. [The Modifier Pattern](#the-modifier-pattern)
3. [Globals](#globals)
4. [State and Memory](#state-and-memory)
5. [Story Card APIs](#story-card-apis)
6. [Info Object](#info-object)
7. [History Array](#history-array)
8. [Sandbox Constraints](#sandbox-constraints)
9. [Return Value Semantics](#return-value-semantics)
10. [Practical Patterns](#practical-patterns)
11. [TypeScript Declarations](#typescript-declarations)

---

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

Plus imperative Story Card functions: `addStoryCard`, `updateStoryCard`, `removeStoryCard`.

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

## Story Card APIs

### addStoryCard(keys, entry?, type?, name?, notes?, options?)

Creates a card with a random ID and pushes it to `storyCards`.

```javascript
const idx = addStoryCard("Amanda,Mandy", "Amanda is a half-elf ranger.", "Character")

// With returnCard option
const card = addStoryCard("Amanda", "Amanda is a ranger.", "Character", "Amanda", "", { returnCard: true })
```

Parameters: `keys` (comma-separated triggers), `entry` (context text), `type` (default "Custom"),
`name` (default: keys), `notes` (default: ""), `options` (`{ returnCard: boolean }`).

### updateStoryCard(index, keys, entry, type?, name?, notes?)

Replaces the card at `index`, preserving its ID. Optional params preserve existing values.

### removeStoryCard(index)

Splices out the card at `index`. Throws if not found. Indices shift after removal.

### Direct Array Manipulation

`storyCards` is a regular mutable array:
```javascript
storyCards.push({ id: "custom-1", keys: ["tavern"], entry: "The tavern is dimly lit." })
storyCards[0].entry = "Updated entry text."
```

---

## Info Object

```javascript
info.actionCount      // total actions in adventure history
info.characterNames   // multiplayer character names

// Only available in onModelContext:
info.maxChars         // approximate max characters for model context
info.memoryLength     // characters used by memory/required-elements section
```

Safe truncation in context scripts: `context.slice(-(info.maxChars - info.memoryLength))`

---

## History Array

```javascript
history[i].text   // the action text
history[i].type   // "start" | "continue" | "do" | "say" | "story" | "see" | "unknown"
```

Most recent action: `history[history.length - 1]`. Contains both player inputs and AI
responses, ordered chronologically.

---

## Sandbox Constraints

- **Memory limit**: 16 MB
- **Execution timeout**: 2 seconds per hook
- **Console logs**: persist 15 minutes, visible only to scenario creator
- **No network access**: no fetch, no XMLHttpRequest, no imports
- **No DOM**: pure JavaScript computation only
- **Optimized Context mode** (DeepSeek V4 Flash, Equinox, Gemma 31B, DeepSeek V4 Pro,
  GLM 5.1) disables certain scripting features in exchange for more context

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

### Dynamic Author's Note (onModelContext)

```javascript
const modifier = (text) => {
  const mood = state.inCombat ? "tense, fast-paced" : "atmospheric, contemplative"
  const an = `[Author's note: ${mood}, second person, present tense.]`
  const lines = text.split("\n")
  lines.splice(-3, 0, an)
  return { text: lines.join("\n") }
}
modifier(text)
```

### Auto Story Card Generation (onOutput)

```javascript
const modifier = (text) => {
  const names = text.match(/\b[A-Z][a-z]+(?:\s[A-Z][a-z]+)*\b/g) || []
  const known = state.knownNames || []
  for (const name of names) {
    if (!known.includes(name) && name.length > 3) {
      addStoryCard(name, `${name} is a character encountered in the story.`, "Character")
      known.push(name)
    }
  }
  state.knownNames = known
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

## TypeScript Declarations

Community-maintained type definitions from the FoxTweaks project
(`github.com/Worldsmythe/FoxTweaks/blob/main/src/aidungeon.d.ts`):

```typescript
interface HookReturn { text?: string; stop?: boolean }

interface History {
  text: string
  rawText?: string
  type: "continue" | "say" | "do" | "story" | "see" | "start" | "unknown"
}

interface StoryCard {
  id: string; keys?: string[]; type?: string; entry?: string
  title?: string; description?: string
  createdAt?: string; updatedAt?: string; deletedAt?: string
  useForCharacterCreation?: boolean
}

interface StateMemory {
  context?: string       // overrides Plot Essentials
  authorsNote?: string   // overrides Author's Note
  frontMemory?: string   // appended to end of context
}

interface State {
  memory: StateMemory
  message: string | ThirdPerson | ThirdPerson[]
  [key: string]: unknown
}

interface Info {
  actionCount: number
  characterNames: Array<string | { name: string }>
  maxChars?: number       // onModelContext only
  memoryLength?: number   // onModelContext only
}

interface ThirdPerson {
  text: string
  visibleTo?: Array<string | { name: string }>
}

// Global functions
function addStoryCard(keys, entry?, type?, name?, notes?, options?): number | StoryCard
function removeStoryCard(index: number): void
function updateStoryCard(index, keys, entry, type?, name?, notes?): void
function log(message: string): void
```
