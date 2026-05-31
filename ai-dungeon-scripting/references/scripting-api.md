# AI Dungeon Scripting API Reference

Exhaustive signatures for the scripting globals: the Story Card functions, the `info`
object, the `history` array, and community TypeScript declarations. The SKILL.md covers the
hooks, modifier pattern, state, and sandbox; this file is the function-level detail.

## Contents
- [Story Card APIs](#story-card-apis)
- [Info Object](#info-object)
- [History Array](#history-array)
- [TypeScript Declarations](#typescript-declarations)

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
