# AI Dungeon Scenario Mechanics

The non-obvious mechanics behind scenario design: branch trees, the three scenario types,
placeholders, story-card fields and trigger matching, and publishing/tags. For *how to write
a good scenario* (the prescriptive guidance and a worked example), see this skill's SKILL.md.

## Table of Contents
- [AI Dungeon Scenario Mechanics](#ai-dungeon-scenario-mechanics)
  - [Table of Contents](#table-of-contents)
  - [Branch Trees](#branch-trees)
  - [Scenario Types](#scenario-types)
    - [Simple Start](#simple-start)
    - [Multiple Choice](#multiple-choice)
    - [Character Creator](#character-creator)
    - [Choosing a type](#choosing-a-type)
  - [Placeholders](#placeholders)
  - [Story Card Fields](#story-card-fields)
  - [Story Card Trigger Words](#story-card-trigger-words)
  - [Publishing, Tags, and Visibility](#publishing-tags-and-visibility)

When a player starts an adventure, the scenario's content is copied in: the Opening becomes
the first action, plus all plot components, story cards, scripts, and AI settings. Author
metadata, comments, and the branch-tree structure itself do not copy — the tree resolves to
the single selected leaf path.

---

## Branch Trees

A scenario is a **tree of branches**. The root is the scenario itself; children are choices
the player navigates from root to leaf. The **leaf** (a branch with no children) is what
actually plays.

- **Leaf branches are playable**: the Opening becomes the first action, all plot components
  enter context, cards trigger, placeholders resolve.
- **Non-leaf branches (root + mid-nodes) are navigation only**: the Opening shows as a
  menu, child titles become buttons, and *nothing else* — plot components, cards,
  placeholders — has any effect.
- **Branches inherit nothing.** Every leaf carries its own complete plot components and
  cards. If two leaves need the same card, each includes it separately. (The `aid mc`
  builder in the `ai-dungeon` skill compiles layered trees so you don't maintain this by
  hand.)

---

## Scenario Types

### Simple Start
Single prompt, no branching. The Opening becomes the adventure's first action. Supports
everything (plot components, placeholders, scripts). The default and most common type.

### Multiple Choice
A branch tree the player navigates before play begins. The root/mid-node Openings frame the
choices; child titles are the buttons; only the selected leaf contributes to the adventure.

```
Fantasy World (root — "Choose your path:")
├── Warrior ("What kind of warrior?")
│   ├── Noble Knight   (leaf — adventure starts here)
│   └── Mercenary      (leaf)
└── Mage
    ├── Court Wizard   (leaf)
    └── Hedge Mage     (leaf)
```

Scripts aren't supported on MC nodes themselves, but a leaf that's Simple Start or Character
Creator can have them. For "each level adds cards/PE" designs, see the `aid mc` layered-tree
builder in the `ai-dungeon` skill (branches don't inherit, so every choice must be baked
into each leaf).

### Character Creator
Formerly "Worlds." A separate AI model **generates the opening dynamically** from cards the
player picks (race, class, faction, etc.), so each playthrough can start differently. You
define the world's options as story cards flagged `useForCharacterCreation`; the card's
`value`/Entry feeds the prompt-generation model, and `description`/Notes is the player-facing
text on the selection screen.

Practical notes:
- Prompt generation is unreliable — it often gets world details wrong or produces a generic
  opening. Use Simple Start or MC when the starting point must be dependable.
- It doesn't preserve the player's picks into Plot Essentials; tell players to copy details
  there (or script it).
- Best for broad sandboxes with lots of race/class/location/faction cards.
- Combine with MC (each branch is a Character Creator) to get "choose era → create character
  within it."

### Choosing a type

| You want | Use |
|----------|-----|
| The same setup every play | Simple Start |
| Different premade starts / paths | Multiple Choice |
| AI-generated openings from a card sandbox | Character Creator |
| Player-defined character/world via questions | Simple Start + Placeholders |

Players generally prefer a "custom/blank-slate" Simple Start option over Character Creator's
generated openings; scenarios offering both tend to see more play on the custom one.

---

## Placeholders

Placeholders prompt the player for input at adventure start; their answer is substituted
everywhere the exact placeholder text appears.

```
You are ${What is your name?}, a ${What is your profession?} in Larion.
→ "You are Marcus, a blacksmith in Larion."
```

- **Where they work:** Opening, Plot Essentials, Author's Note, AI Instructions, Story
  Summary, and all story-card fields. They do **not** resolve on non-leaf MC branches.
- **Exact-match reuse:** identical placeholder text is asked once and filled everywhere. Any
  difference in spelling/case/punctuation makes a separate prompt: `${Name}` ≠ `${name}`.
- **Skippable for "AI decides":** a placeholder left blank resolves to empty (in the Opening
  it stays literal `${...}`). Put one last in the Opening to let the player hand a choice to
  the AI: `threatened by ${What threatens the world? (blank = AI decides)}`.
- **Nesting:** a placeholder inside another resolves inner-first, substituting the answer
  into the outer question. Powerful for conversational flows, but costs characters and
  confuses players past a couple levels.
- **Built-in character placeholders** auto-handle name, gender, and pronouns:
  `${character.name}`, `${character.gender}` (Male/Female/Custom), and
  `${character.pronoun.they|them|their|theirs|themselves}` (auto-filled from the gender
  choice — no extra prompt).

**Placeholder-mirror trick:** for a player-built character/world, duplicate the Opening's
`${...}` text into Plot Essentials. The Opening creates the scene; the PE copy keeps the
answers in always-on context after the Opening scrolls out of history (otherwise the AI
forgets who the player defined). This is the backbone of the placeholder-template shape.

Keep it to roughly 3–6 meaningful, character-defining questions. One "Describe your
appearance" beats six asking hair/eye/height separately.

---

## Story Card Fields

```json
{
  "keys": "Elena, princess",
  "value": "Elena is the crown princess of Valtara, sharp-tongued and loyal.",
  "type": "character",
  "title": "Elena",
  "description": "",
  "useForCharacterCreation": false
}
```

| Field | AI sees it? | Purpose |
|-------|-------------|---------|
| `keys` | no (controls firing) | comma-separated trigger words |
| `value` | **yes** | the entry text injected when triggered |
| `type` | no | your own category label (`character`, `location`, `Custom`, …) |
| `title` | no | editor display name |
| `description` | no in play | author notes; player-facing in Character Creator |
| `useForCharacterCreation` | n/a | surfaces the card as a Character Creator option |

- **`value` vs `entry`:** same field, two names — `value` in the GraphQL/JSON, `entry` in the
  scripting API and UI.
- **Repeat the subject's name in `value`.** The AI sees the entry, never the title; a card
  titled "Elena" whose value opens "She is…" gives no anchor.
- **Cards are world reference, not plot.** Entries describe what *exists* (people, places,
  factions, items, lore) and fire on a keyword. They are not hooks, quests, or premise —
  those go in the Opening and Plot Essentials. Test: reads like a wiki entry → card; a
  situation you want the story to start from or move toward → not a card.
- **`type` is for you, not the AI** — the AI never sees it. `Custom` is common.

**Taxonomy cards** (for a concept with sub-types): give the concept card a broad trigger and
each sub-type card *both* its own name and the parent term, so a mention pulls the right
depth. E.g. a `Quirks` card (keys `Quirks, superhuman ability`) plus `Emitter` (keys
`Emitter, quirks`) and `Transformation` (keys `Transformation, quirks`) — naming a sub-type
fires its detail card *and* the concept card.

For the full GraphQL type and the markdown authoring format, see the `ai-dungeon` skill
(GraphQL API reference → "StoryCard Object"; and the `aid convert` command).

---

## Story Card Trigger Words

A trigger matches as a **substring of recent text**, case-insensitively. Spaces around it
control greediness, and this is the main thing to get right.

| Trigger form | Matches | Risk |
|--------------|---------|------|
| `elf` | "elf", "shelf", "self", "elves" | bleeds into longer words |
| ` elf` (leading space) | " elf", not "shelf" | won't fire at line start or after a quote |
| `elf ` (trailing space) | "elf ", not "elfin" | won't fire before punctuation |
| ` elf ` (both) | only " elf " | most precise, most fragile |

- **Short/common triggers need guards.** Bare `elf` fires inside "shelf"/"yourself". For a
  leading-space trigger that should also catch line starts and quotes, add variants:
  ` elf,"elf,'elf`.
- **Plural "s" is automatic** (`frisbee` catches `frisbees`) *unless* you added a trailing
  space. Irregular plurals are not — add `elves` separately.
- **Longer phrases avoid bleed.** `bronze dragon` won't fire on "Yellow Dragon Inn".
- **Hyphens/symbols split tokens**, so ` Dragon` (leading space) won't fire on "Yellow-Dragon".
- **AI-written triggers fire next turn**, not the current one; player-typed triggers fire
  immediately. (This is also how you pull a card in deliberately mid-play — name its key.)
- **Stub to a root** instead of listing variants: `therap` covers therapy/therapist/
  therapeutic in one trigger, saving the 100-char key budget.

`aid keys "<word>"` (in the `ai-dungeon` skill) generates a correctly guarded trigger set
sized to the word's length.

---

## Publishing, Tags, and Visibility

**Visibility:** Private (only you), Unlisted (anyone with the link, not searchable),
Published (public + searchable, gated behind an AI-moderation review). Content ratings:
Everyone / Teen / Mature / Unrated; NSFW content needs the flag.

**Tags** are the main discovery lever:
- Lowercase, human-readable. Spaces are fine in modern AI Dungeon — `slice of life` is one
  tag, not two. Skip emoji (those go in the title/description).
- Don't spend a tag on the default. Second-person present POV is the norm, so `second person`
  is a wasted slot — tag what's distinctive instead.
- **10 maximum**, hard limit. Spend them across genre / theme / setting / tone / style /
  feature rather than stacking near-synonyms. (`aid tags` lints a list.)

**Description** is player-facing only — the AI never sees it. Lead with a short pitch; put
credits, scripts-used, and version notes below. Script credit markers like `[IS🎭]` (Inner
Self), `[🎴]` (Auto-Cards), `[🦊]` (FoxTweaks) go in the **title/description**, never on cards
and never in the tag field.

Editing a published scenario applies immediately to new adventures; existing adventures keep
their state.
