---
name: ai-dungeon
description: >
  Reference guide for AI Dungeon gameplay concepts, scenario design, and scripting.
  Use this skill whenever the user mentions AI Dungeon, AID, story cards, author's note
  (in an AI Dungeon context), plot essentials, Do/Say/Story/Continue actions, AI Dungeon
  scenarios, AI Dungeon scripting, AI Dungeon adventures, or wants to brainstorm, design,
  or debug an AI Dungeon setting, scenario, or script. Also trigger when discussing
  AI Dungeon tier limits (Wanderer/Champion/Legend/Mythic), AI Dungeon models, or
  context window budgeting for interactive fiction on AI Dungeon. If the user says
  "let's brainstorm an AI Dungeon setting" or "help me design a scenario," this skill
  has everything you need.
---

# AI Dungeon — Gameplay, Scenario Design & Scripting Reference

This skill covers AI Dungeon's Phoenix-era platform (play.aidungeon.com / beta.aidungeon.com).
Use it to brainstorm settings, design scenarios, write story cards, craft author's notes,
understand context budgets, and work with the scripting system.

**Reference files** (read on demand):
- `references/scenario-design.md` — Scenario structure, branch trees, scenario types (Simple Start, MC, Character Creator), placeholders, publishing
- `references/scenario-patterns.md` — What works: popular scenario analysis, plot component strategies, community scripts, best practices, design workflow
- `references/gameplay.md` — Playing well: coherence, plot component management mid-adventure, troubleshooting
- `references/scripting.md` — Scripting API: hooks, state, memory overrides, story card functions, code patterns
- `references/graphql-api.md` — GraphQL endpoint, auth, key queries, content model, search/discovery

---

## How AI Dungeon Works

AI Dungeon is an LLM-powered interactive fiction platform by Latitude. A **Scenario** is a
reusable template (prompt, story cards, scripts, plot components). An **Adventure** is a
unique playthrough created from a scenario. Each turn, the player submits an action, the
system assembles a context window from layered components, sends it to the model, and
displays the response.

AID is not a single-model system. The platform uses multiple AI models for different tasks:
a story generation model, a memory formation model, a summarization model, a
character-creator opening prompt model, a story card entry generation model, an embeddings
model for memory relevance scoring, image generation models, safety/moderation models
(HIVE), and Claude for content rating determination. Not all of these fire every turn —
it depends on the action and configuration.

Scenarios have two identifiers: a `shortId` (URL-visible, shareable) and an internal `id` (UUID).

---

## Context Assembly Order

Every turn, AI Dungeon builds the model context in this order:

```
 1. AI Instructions        — system prompt (behavioral rules, style, POV)
 2. Plot Essentials         — always-on key facts
 3. World Lore: [Cards]     — keyword-triggered story card entries
 4. Story Summary           — auto-compressed narrative arc
 5. Memories                — vector-retrieved Memory Bank entries
 6. Recent Story: [History] — recent actions and responses
 7. [Author's note: ...]    — ~3 lines from end, in brackets
 8. Last Action             — most recent player input or AI output
 9. Front Memory            — scripting-only, appended at very end
10. Buffer Tokens           — reserved space for AI response
```

### Budget Split

**Required Elements** (AI Instructions, Plot Essentials, Story Summary, Author's Note,
Front Memory, Last Action) get up to **70%** of total context. When they compete for space,
priority is: Front Memory & Last Action (always full) > Author's Note > Plot Essentials >
AI Instructions > Story Summary.

**Dynamic Elements** (~30% remaining) split roughly: ~25% Story Cards, ~50% History,
~25% Memory Bank. If Memory Bank is disabled, its share goes to History.

Content at the beginning and end of context gets the most attention from the model.
Plot Essentials and Author's Note are therefore the strongest steering tools.

---

## Action Modes

| Mode         | What the AI sees                                  | Use for                                    |
|--------------|---------------------------------------------------|--------------------------------------------|
| **Do**       | `> You [text]` (pronouns flipped to 2nd person)   | Character actions                          |
| **Say**      | `> You say, "[text]"` (auto-quoted)                | Dialogue                                   |
| **Story**    | Raw text, no `>` prefix, no pronoun changes        | Narration, scene-setting, other POVs       |
| **Continue** | No input — model extends the story                 | Let the AI keep writing                    |
| **See**      | Image generation prompt (costs Credits)            | Visualize a scene                          |

**Retry** re-rolls the latest output. **Erase** removes it. **Edit** rewrites any prior
action inline. Edit + Continue is the canonical technique for course-correcting a story.

Overusing Do/Say causes repetitive `> You...` patterns. Mix Story mode to break the rhythm.
The Wayfarer model line reads `>` as input; Hermes 3 reacts poorly to `>` and works better
with Story/Edit.

---

## Plot Components

Every scenario branch has 5 text fields. **These only affect AI behavior on leaf branches**
(see `references/scenario-design.md`). On non-leaf branches, only the Opening is shown.

### AI Instructions
System prompt at the very beginning of context. Put behavioral rules, writing style, POV,
things to avoid here. Custom instructions replace defaults (they don't layer). This is where
80% of steering belongs. Stored internally as `{ type: "scenario", scenario: "text" }`.

### Plot Essentials (formerly "Memory")
Always-on key facts: protagonist traits, companions, goals, world rules. Name the subject
on every line. Avoid negatives ("don't mention X" fails — say "avoid X" or state what IS true).
The most powerful tool for persistent world knowledge.

### Story Summary
Running summary of plot and world state. Auto-maintained every ~15 actions if Auto
Summarization is enabled. Manual edits feed back into subsequent auto-summarization. In
scenarios, use this for backstory the AI should know. Often best left empty unless needed.

### Author's Note
Injected ~3 lines from the end in brackets: `[Author's note: ...]`. Controls tone, genre,
style, perspective. Keep it to 3-4 sentences max (~150 chars is the practical sweet spot).
Its power comes from late position, not length. Common format:
`Genre: dark fantasy; Tone: ominous; Focus: the approaching storm`

**AI Instructions vs Author's Note**: AI Instructions = general rules at top of context.
Author's Note = the few instructions you want maximally weighted this turn.

### Third Person
Boolean toggle. Converts "You" in Do/Say actions to the character's name. Primarily for
multiplayer or named protagonists.

---

## Story Cards

Story Cards (formerly "World Info") are keyword-triggered context injections. They solve
the "AI forgets things" problem for entities that aren't always relevant.

### Card Fields

| Field        | AI sees it? | Purpose                                              |
|--------------|-------------|------------------------------------------------------|
| **Type**     | No          | Category (Character, Class, Race, Location, etc.)    |
| **Name**     | No          | Your reference label. **AI cannot see this.**        |
| **Entry**    | Yes         | Text injected as `World Lore:` when card triggers    |
| **Triggers** | No          | Comma-separated keywords that activate the card      |
| **Notes**    | No          | Your memos; shown as description in Character Creator|

### Trigger Mechanics
- Case-insensitive, whitespace-sensitive (`Amanda` ≠ ` Amanda`)
- Substring matching: `dragon` catches `dragons` but also `dragonfly` — use specific terms
- Scans minimum 4 recent actions; more with available budget (~1 extra per 100 tokens)
- Matched cards sorted by recency/frequency, injected up to ~25% of dynamic budget
- A trigger inside an AI response won't activate its card until the next turn
- Cap: 5,000 cards per scenario/adventure

### Key Practices
- **Repeat the name inside Entry text.** The AI can't see the Name field.
- **Use proper nouns as triggers** to avoid false positives.
- **Cross-reference between cards.** "Amanda's friend is Jeremy" makes triggering one cascade.
- **Keep entries concise** (2-4 sentences). They're first dropped when context overflows.
- **Embed key names in Plot Essentials** to prime the AI, which triggers their cards.

### Story Cards + Author's Note
Cards supply **what exists**. Author's Note tells the model **how to render it**. Same
castle card with `[Horror, slow burn dread]` produces something completely different than
`[Lighthearted tour-guide tone]`. Author's Note is protected when context is tight; cards
drop first.

---

## Memory System

Three components:

1. **Story Summary** — auto-generated every ~15 actions, captures narrative arc
2. **Memory Bank** — vector-embedded summaries of every ~6 actions, ranked by similarity
   to the current action. Slots: 25 (Free) / 100 / 200 / 400
3. **Auto Summarization** — the process that generates both of the above

Enable via Gameplay → AI Models → Memory System. On the free tier with 25 slots, this is
the single highest-leverage upgrade.

---

## Tier Limits

### Wanderer (Free) — What You Get
All plot components, all story card features (5,000 cap), Memory Bank + Auto Summarization
(25 slots), full scripting editor, multiplayer, all action modes, See mode (requires
purchased Credits). Limited daily Premium Actions on Dynamic Large.

### What's Gated

| Feature            | Wanderer | Champion ($15/mo) | Legend ($30/mo) | Mythic ($50/mo) |
|--------------------|----------|--------------------|-----------------|-----------------|
| Context tokens     | 4K       | 8K                 | 16K             | 32K             |
| Memory Bank slots  | 25       | 100                | 200             | 400             |
| Monthly Credits    | 0        | 760                | 1,650           | 2,750           |

Shadow Tiers (Wraith/Banshee/Reaper/Apocalypse) sit above Mythic, doubling context per step.
Some models support Credit-per-action context extension up to 128K.

Free models (as of mid-2026, changes frequently): Dynamic Small, Wayfarer Small 2, Muse 12B,
Madness, Fable, DeepSeek V4 Flash, Hearthfire, Harbinger. 2K context on heavier free models.

### Upgrade Decision Points
- Adventures regularly exceed ~50 actions, AI loses thread → Champion ($15)
- You specifically benefit from premium models → Legend ($30)
- Mythic is "NOT the right tier for most players" per Latitude

---

## Reference Files

| File | When to read |
|------|-------------|
| `references/scenario-design.md` | Scenario structure, branch trees, leaf vs non-leaf, scenario types (Simple Start, Multiple Choice, Character Creator), placeholders, publishing |
| `references/scenario-patterns.md` | Popular scenario analysis, plot component strategies from real data, community scripts (Auto-Cards, Inner Self), best practices, design workflow |
| `references/gameplay.md` | Keeping the AI coherent, managing plot components during play, scene transitions, troubleshooting common problems |
| `references/scripting.md` | Full scripting API: hooks, state, memory overrides, story card APIs, sandbox limits, code patterns |
| `references/graphql-api.md` | GraphQL endpoint, auth, key queries (search, scenario details, recently played), content model |
