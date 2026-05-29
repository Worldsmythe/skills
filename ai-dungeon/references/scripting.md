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
11. [Installing and Combining Scripts](#installing-and-combining-scripts)
12. [Script Catalog and Compatibility](#script-catalog-and-compatibility)
13. [TypeScript Declarations](#typescript-declarations)

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
- **Optimized Context / `cacheEfficient` mode** (DeepSeek V4 Flash, Equinox, Gemma 31B,
  DeepSeek V4 Pro, GLM 5.1) caches the prompt. The `onModelContext` modifier still **runs**,
  so *reading* the context (to detect mentions, update `state`, score sentiment) works — but
  any **change it writes to the context text is discarded**. Practically: a feature that
  injects instructions/thoughts/cards/stats by returning modified context text is
  **non-functional** on these models. Payloads delivered through `state.memory.frontMemory`
  or `state.memory.authorsNote` (set from any hook, not the returned context text) survive,
  as do `onInput`/`onOutput` features. The catalog flags this per script.

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

## Installing and Combining Scripts

Scripts are community code. **Get them from the canonical repo and upload them as-is —
never hand-write or "recreate" a script, and never reconstruct one by reading the
scenario's current scripts and editing in place.** Clone the source, then push it.

**Single script.** Clone and push the four files to the four slots:

```bash
git clone --depth 1 https://github.com/LewdLeah/Inner-Self /tmp/inner-self
aid scripts <shortId> --scripts-dir /tmp/inner-self/src
```

`aid scripts --scripts-dir` maps `input.js`/`output.js`/`context.js`/`library.js` → the
Input/Output/Context/Library slots and *merges* (slots you don't supply are left alone).
Layouts vary — some repos keep the files at the root, ship a built bundle on a `dist`
branch (FoxTweaks), or use `.txt` names with spaces (UBIS, SAE, TAS). When names don't
match, pass slots explicitly (`--shared-library @file --on-input @file …`) or stage them
into a temp `src/` first.

**Combining scripts.** You don't need a "compatible" rewrite — read each script's
Input/Context/Output and fold them into one ordered modifier per slot, with every script's
library concatenated in Library. Each catalog entry below gives the per-slot hook call. The
standard shape:

```javascript
// Input slot — Inner Self, then FoxTweaks
InnerSelf("input");
const modifier = (text) => {
  text = FoxTweaks.Hooks.onInput(text);
  return { text };
};
modifier(text);
```

Order matters only when two scripts touch the same thing (the context string, the Author's
Note, the same story card). The table flags those; reconcile only there.

**`cacheEfficient` / Optimized-Context models.** The Context modifier still runs (reading
context is fine), but **any change it writes to the context text is discarded**, so a
feature that injects content by returning modified context is non-functional. At a glance:

- **Non-functional:** Inner Self, Auto-Cards, Localized-Languages, Story Arc Engine, True
  Auto Stats, MindForge (all inject thoughts/cards/arc/stats/brains via the context text),
  and UBIS (its world-state block is a context append).
- **Partial:** FoxTweaks (only Random Names' name-bank injection is lost; everything else is
  input/output). CSMS (character sheets delivered via `state.memory.frontMemory` survive;
  in-context roll prompts are dropped).
- **Functional:** Slowburn (no Context hook; writes the Author's Note from input/output).

Payloads sent via `state.memory.frontMemory` / `state.memory.authorsNote` survive because
they aren't the returned context text. (Exact platform boundary unverified on a live model
— treat this as "context-text injection won't reach the model.")

## Script Catalog and Compatibility

Community scripts the skill knows about. **Clone from the repo; don't recreate.** Each entry
gives the per-slot chain call (for combining) and notes which feature(s) rely on rewriting
context (and so go inert on `cacheEfficient` models).

### Inner Self — `github.com/LewdLeah/Inner-Self`
Persistent NPC "minds" — memory, goals, secrets, self-reflection in per-character brain
cards. The modern standard, and it **bundles Auto-Cards (don't also install Auto-Cards).**
- **Chain:** `InnerSelf("input"|"context"|"output");` at the top of each slot's modifier; Library = its full `library.js`.
- **cacheEfficient:** not safe — the Context hook injects NPC brains every turn, so the script is inert on Optimized-Context models.
- **State/cards:** `state.InnerSelf` (+ shared `AutoCards`/`LSIv2` keys); cards `Configure \nInner Self`, `@`-prefixed brains.

### Auto-Cards — *deprecated* — `github.com/LewdLeah/Auto-Cards`
Auto-generates/updates story cards from play ("object permanence"). **Superseded by Inner
Self, which embeds it — prefer Inner Self for new scenarios; don't install both.**
- **Chain:** `AutoCards("input"|"context"|"output", text)` in each slot.
- **cacheEfficient:** **non-functional** — card generation injects its prompt by rewriting the context, which is discarded, so no cards are generated.
- **State/cards:** `state.AutoCards`/`LSIv2`; cards `Configure \nAuto-Cards`, `Debug Data`.

### Localized-Languages (LoLa) — `github.com/LewdLeah/Localized-Languages`
Translates gameplay into ~260 languages. Ships **two variants**: with bundled Auto-Cards
(`src/`) and **without** (`src (Without Auto-Cards)/`) — use the no-AC variant beside Inner
Self / Auto-Cards.
- **Chain:** `LocalizedLanguages("input"|"context"|"output", text)`; runs *after* Auto-Cards in each slot.
- **cacheEfficient:** not safe — Context rewrites the whole context (renames sections, re-truncates).
- **Setup:** requires `{Language: ${Select your language:}}` in the Opening.
- **State/cards:** `state.LocalizedLanguages` (+ shared AC keys in the full variant); card `Localized Languages`.

### FoxTweaks — `github.com/Worldsmythe/FoxTweaks`
Config-card plugin bundle: dice rolls, paragraph formatting, redundancy merge, pronoun
fixes, random names, `{{placeholders}}`. **Built to wrap Inner Self / Auto-Cards.**
- **Chain:** `text = FoxTweaks.Hooks.onInput(text)` (and `onContext`/`onOutput`) inside each slot's modifier; Library = the built `dist` bundle `foxtweaks.js`.
- **cacheEfficient:** mostly safe — only **Random Names' name-bank injection** (into the Author's Note, via Context) is skipped; its output-side name *replacement* and every other feature (dice = input; paragraph/redundancy/pronoun = output) still work.
- **State/cards:** namespaced under `state.foxTweaks`; card `FoxTweaks Config`.

### Story Arc Engine (SAE) — `github.com/Yi1i1i/Story-Arc-Engine`
Periodically AI-generates an 11-beat arc and injects it into the Author's Note for
long-range coherence.
- **Chain:** `onInput_SAE` / `onContext_SAE` / `onOutput_SAE` (ships as standalone modifiers — fold them into your chain).
- **cacheEfficient:** not safe — Context strips `<<…>>` and rewrites the `[Author's note]`.
- **State/cards:** **un-namespaced** top-level `state` keys (`storyArc`, `arcPrompt`, `turnNum_SAE`, …) — watch for collisions; cards `Current Story Arc`, `Story Arc Settings`. Writes the Author's Note.

### True Auto Stats (TAS) — `github.com/Yi1i1i/True-Auto-Stats-for-AIDungeon-RPG-Scenarios`
Auto-creates/updates RPG stats, inventory, skills from natural play.
- **Chain:** `onInput_TAS` / `onContext_TAS` / `onOutput_TAS` (standalone modifiers — fold in).
- **cacheEfficient:** not safe — Context splices `[Assume player…]` stat blocks mid-context.
- **State/cards:** bare `state` keys; `${player} Stats/Inventory/Skills/…` cards in a "Player Stats" group.

### Character Sheet & Mechanics (CSMS) — `github.com/NikolaiF90/AIDCharacterSheetandMechanicSystem`
D&D-style sheets, dice/stat checks, combat, inventory, XP/leveling.
- **Chain:** `CSMS("input"|"context"|"output");` then your modifier (chain-friendly by design).
- **cacheEfficient:** **partial** — character sheets go through `state.memory.frontMemory` (survives), but in-context roll/check prompts added via the context hook are discarded.
- **State/cards:** namespaced (`csms*`); cards `⚙️ CSMS CFG`, per-character `📋 {name}`.

### Ultimate Banking & Inventory (UBIS) — `github.com/Itsbrazyyy/Ultimate-Banking-Inventory-System`
Currency, inventory, time, bills/income, weather, holidays/events.
- **Chain:** `onInput_UBIS()` / `onContext_UBIS()` / `onOutput_UBIS()` (standalone modifiers — fold in).
- **cacheEfficient:** **non-functional** — its `[CURRENT WORLD STATE]` block is a context append (`return text+ws`), which is discarded, so the model never sees currency/inventory/time. (Bookkeeping in input/output still runs.)
- **State/cards:** `state.ubis`; 10 cards in a "UBIS" group.

### Slowburn — `github.com/saya-evrlc/Slowburn`
Tracks one NPC's 0–100 "evolution" from output sentiment and injects the stage into the
Author's Note. No Context hook.
- **Chain:** `SLOWBURN("input", text)` / `SLOWBURN("output", text)` — built to paste *below* other scripts.
- **cacheEfficient:** **functional** — no Context hook; it sets the Author's Note via `state.memory.authorsNote` from input/output, outside the discarded context-text path.
- **State/cards:** `state.npc`, `state.memory.authorsNote` (writes the Author's Note); reads cards containing "Evolution Stages".

### MindForge — `github.com/Leolynns/MindForge`
Per-NPC private-memory engine (brain cards + a hidden memory-op prompt). **An alternative to
Inner Self — don't run both.**
- **Chain:** `MindForge("input"|"context"|"output");` (ships as standalone modifiers — fold in).
- **cacheEfficient:** not safe — Context injects brains + a `<SYSTEM>` block and trims Recent Story.
- **State/cards:** `state.MindForge`; card `Configure MindForge`, per-NPC brain cards.

### Compatibility at a glance

Out-of-the-box behavior, before any hand-merge. Most pairs are **mergeable** — read each
script's slots and fold them into one ordered chain; the marks flag what to reconcile.

| | IS | AC | LoLa | FT | SAE | TAS | CSMS | UBIS | SB | MF |
|---|---|---|---|---|---|---|---|---|---|---|
| **Inner Self** | — | ✗ᵃ | ◆ᵇ | ✓ᵈ | ◆ | ◆ | ✓ | ◆ | ✓ | ✗ᶜ |
| **Auto-Cards** | ✗ᵃ | — | ◆ᵇ | ✓ᵈ | ◆ | ◆ | ✓ | ◆ | ✓ | ◆ |
| **Localized-Lang** | ◆ᵇ | ◆ᵇ | — | ◆ | ◆ | ◆ | ✓ | ◆ | ✓ | ◆ |
| **FoxTweaks** | ✓ᵈ | ✓ᵈ | ◆ | — | ◆ | ◆ | ✓ | ◆ | ✓ | ◆ |
| **Story Arc** | ◆ | ◆ | ◆ | ◆ | — | ◆ | ◆ | ◆ | ◆ᶠ | ◆ |
| **True Auto Stats** | ◆ | ◆ | ◆ | ◆ | ◆ | — | ✗ᵍ | ◆ | ◆ | ◆ |
| **CSMS** | ✓ | ✓ | ✓ | ✓ | ◆ | ✗ᵍ | — | ◆ | ✓ | ◆ |
| **UBIS** | ◆ | ◆ | ◆ | ◆ | ◆ | ◆ | ◆ | — | ◆ | ◆ |
| **Slowburn** | ✓ | ✓ | ✓ | ✓ | ◆ᶠ | ◆ | ✓ | ◆ | — | ◆ |
| **MindForge** | ✗ᶜ | ◆ | ◆ | ◆ | ◆ | ◆ | ◆ | ◆ | ◆ | — |

✓ drop-in chainable · ◆ mergeable — fold the slots into one ordered chain · ✗ don't combine (redundant/duplicate)

- **ᵃ** Inner Self bundles Auto-Cards — never install both.
- **ᵇ** Use Localized-Languages' "Without Auto-Cards" variant beside Inner Self / Auto-Cards (else Auto-Cards is bundled twice); both rewrite context, so order them.
- **ᶜ** Inner Self and MindForge are both per-NPC mind engines — pick one.
- **ᵈ** FoxTweaks is built to wrap Inner Self / Auto-Cards (its README shows the order).
- **ᶠ** Both write the Author's Note — have one yield, or merge the note text.
- **ᵍ** True Auto Stats and CSMS are both character/stat systems sharing player-stat cards — pick one.

A ◆ between two context-rewriters (Inner Self, LoLa, FoxTweaks, SAE, TAS, MindForge) means
order their context edits so they don't clobber each other — and on `cacheEfficient` models
those context features are inert anyway.

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
