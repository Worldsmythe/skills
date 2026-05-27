# AI Dungeon Scenario Design Reference

How scenarios are structured, the three scenario types, branch trees, placeholders,
and publishing. For what actually works in practice, see `scenario-patterns.md`.

## Table of Contents
1. [Scenario Structure](#scenario-structure)
2. [Branch Trees: Leaf vs Non-Leaf](#branch-trees)
3. [Scenario Types](#scenario-types)
4. [Placeholders](#placeholders)
5. [Publishing and Visibility](#publishing-and-visibility)
## Scenario Structure

A Scenario is a reusable template. When a player "plays" a scenario, a new Adventure is
created with the scenario's content. Scenarios are authored at URLs like
`/scenario/{shortId}/{slug}/edit`.

### What Transfers to Adventures

**Copied**: Prompt (as first action with type "start"), all Plot Components, Story Cards,
Scripts, Title/Description as defaults, AI settings.

**Not copied**: Scenario metadata (author info, play counts), comments, the branch tree
structure itself (resolved to the single selected leaf path).

### Scenario Components

| Component        | Purpose                                              |
|------------------|------------------------------------------------------|
| Title            | Displayed in search and discovery                    |
| Description      | Player-facing info (AI does NOT see this)            |
| Prompt (Opening) | First text of the adventure on leaf branches         |
| Plot Components  | AI Instructions, Plot Essentials, Story Summary, Author's Note, Third Person |
| Story Cards      | Keyword-triggered lore/character/location entries    |
| Scripts          | JavaScript hooks (onInput, onModelContext, onOutput) |
| Tags             | Keywords for discoverability                         |
| Configuration    | Visibility, NSFW flag, comments, third person default|

---

## Branch Trees

A scenario is a **tree of branches**. The root branch is the scenario itself; child
branches are choices the player navigates. Branches can nest arbitrarily deep.

When a player starts an adventure, they navigate from root to leaf. The **leaf** (a branch
with no children) becomes their starting point.

### Leaf vs Non-Leaf — The Critical Distinction

**Leaf branches** (no children) are **playable**:
- The Opening becomes the first text of the adventure
- All plot components enter the AI's context
- Story cards are active and trigger normally
- Placeholders resolve and prompt the player for input
- This is where the actual adventure begins

**Non-leaf branches** (root + mid-tree nodes) are **navigation only**:
- The Opening is displayed as a question/menu
- Child branch titles appear as clickable buttons below the Opening
- **No other plot components enter the AI's context — they are completely ignored**
- Placeholders do NOT resolve on non-leaf branches
- AI Instructions, Plot Essentials, Story Cards, etc. on non-leaf branches have zero effect

**Branches are self-contained.** A child inherits nothing from its parent. Every leaf
carries its own complete set of plot components and story cards. If multiple leaves need the
same Story Cards, each leaf must include them separately.

---

## Scenario Types

### Simple Start

Single prompt, no branching. The scenario's Opening becomes the adventure's first text.
Supports scripting, placeholders, and all plot components. The simplest and most common type.

### Multiple Choice

Creates a branching setup flow using the branch tree structure. Players navigate through
options before gameplay begins.

```
Fantasy World (Root — non-leaf, Opening = "Choose your path:")
├── Warrior (mid-node — Opening = "What kind of warrior?")
│   ├── Noble Knight  (leaf — full scenario, adventure starts here)
│   └── Mercenary     (leaf — full scenario, adventure starts here)
├── Mage (mid-node)
│   ├── Court Wizard  (leaf)
│   └── Hedge Mage    (leaf)
└── Rogue (mid-node)
    ├── Thief         (leaf)
    └── Assassin      (leaf)
```

**Key rules**:
- Parent scenario's Opening introduces/frames the choices; child titles become buttons
- Only the final selected leaf contributes to the adventure
- Parent scenarios contribute **nothing** to the final adventure
- No automatic content inheritance — each leaf is fully self-contained
- Scripts are NOT supported on Multiple Choice scenario nodes themselves, but their
  child leaf options CAN have scripts (if those leaves are Simple Start or Character Creator)

**Design patterns**:
- Genre → Character (L1: choose genre, L2: choose character)
- Setting → Role → Details (3 levels of narrowing)
- Simple Two-Level (L1: introduce concept, L2: start adventure)

### Character Creator

Formerly called "Worlds." Character Creator is fundamentally different from the other two
types: **an AI model generates the opening prompt dynamically** based on the player's
selections. The player picks from card-based options (race, class, faction, etc.), and a
separate "character-creator opening prompt model" synthesizes those choices into a starting
scenario. This means every playthrough can start differently even with the same selections.

Think of it as building a world sandbox: you define the world's races, classes, locations,
and factions as Story Cards, the player picks from those menus, and the AI writes an
opening that incorporates their choices.

**Setup**:
1. In the scenario editor, go to Plot → Opening Story → click the gear icon → choose
   Character Creator
2. Create Story Cards with appropriate Types (Character, Class, Race, Location, Faction,
   or custom fields)
3. To add existing cards: go to Story Cards → click `...` on a card → "Add to Character
   Creator." You must set the card's `type` to match the field you're adding it to.
4. Custom fields can be added at the bottom of the Character Creator panel, below Factions

**How the card fields map**:
- **Type**: determines which selection step/field the card appears in (Character, Class,
  Race, Location, Faction, or your custom fields)
- **Entry** (called `value` in GraphQL): AI-facing info fed to the prompt generation model
- **Notes** (`description` in GraphQL): player-facing text shown on the selection screen
- **Name** (`title`): the label shown to the player alongside Notes
- **Triggers**: still function normally during play after the adventure starts

**Example**:
```
Card:
  Type: Character
  Name: The Wandering Knight
  Entry: "Sir Marcus is a wandering knight who abandoned his post after
    witnessing corruption in the royal court. He carries a blessed blade
    named Dawnbringer and seeks to restore his honor."
  Triggers: "Marcus, Sir Marcus, the knight, Dawnbringer"
  Notes: "A disgraced knight seeking redemption. Strong in combat but
    haunted by past failures."

Player sees: "A disgraced knight seeking redemption..."
AI prompt model receives: the Entry text + other selected cards → generates opening.
```

**Known issues and limitations** (as of mid-2026):
- The prompt generation AI frequently gets world details wrong — e.g., a "Bioweapon" race
  might produce a story about an ordinary person fighting for justice
- The AI may forget which character the player created, since Character Creator doesn't
  automatically add player info to Plot Essentials the way placeholders do. Players must
  manually add their character info to Plot Essentials.
- The generated prompt appends content after your Opening prompt — you can't prevent this,
  it's by design
- Placeholders (`${}`) are now supported in Character Creator mode (this was added after
  the original launch — older guides may say otherwise)
- The "Quickstart" button creates a random combination from all fields, which can produce
  nonsensical results. Advise players to avoid it in your description.
- The prompt generation quality is widely considered unreliable by the community

**Practical advice for creators**:
- Add a note to your description telling players to copy their character info into Plot
  Essentials manually, since the AI will eventually forget
- Consider adding a Plot Essential that explains this to new players (and asks them to
  delete the note after reading)
- Character Creator works best for **world sandboxes** with lots of content in classes,
  races, locations, and factions — scenarios where you want the AI to take the story
  wherever it wants based on the player's choices
- If you want a specific plotline or reliable starting point, use Simple Start or MC instead

**Combining with Multiple Choice**: use MC branches where each branch is a Character
Creator scenario. This gives structure (choose era → then create character within that era)
while still getting dynamic prompt generation at the leaf level.

Supports scripting (onInput, onModelContext, onOutput).

### When to Use Which Type

| You want... | Use |
|-------------|-----|
| A specific story with the same setup every time | **Simple Start** |
| Same plot, but different starting characters/paths | **Multiple Choice** |
| A world sandbox where the AI generates the opening | **Character Creator** |
| Maximum player customization via questions | **Simple Start + Placeholders** |
| Script-driven experience (Auto-Cards, Inner Self) | **Simple Start** |

Community observation: players tend to prefer the freedom of Simple Start "custom/blank
slate" options over Character Creator's AI-generated openings. Scenarios offering both
MC premade paths and a "custom" Simple Start option see higher actions-per-play on the
custom option.

---

## Placeholders

Placeholders prompt players for input at adventure start. Player answers replace the
placeholder text everywhere it appears. They work in Story, Multiple Choice, and
Character Creator scenarios.

### Syntax
```
${Question text?}
```

### Example
```
Prompt: "You are ${What is your name?}, a ${What is your profession?}
in the kingdom of Larion."

Player enters: "Marcus" and "blacksmith"

Result: "You are Marcus, a blacksmith in the kingdom of Larion."
```

### Where Placeholders Work

| Location                 | Works? |
|--------------------------|--------|
| Opening (Prompt)         | Yes    |
| Plot Essentials          | Yes    |
| Author's Note            | Yes    |
| AI Instructions          | Yes (added March 2026) |
| Story Summary            | Yes (added March 2026) |
| Story Card Entry         | Yes    |
| Story Card Name          | Yes    |
| Story Card Trigger       | Yes    |
| Story Card Notes         | Yes    |
| Character Creator        | Yes (added post-launch) |
| Non-leaf MC branches     | **No** (don't resolve) |

### Resolution Order
Placeholders are asked in order as they appear, across components in this sequence:
1. Plot Essentials
2. Opening Prompt
3. Author's Note
4. Story Cards (bottom of list first, working upward to most recent)
5. Story Card sub-fields: Entry → Name → Trigger → Notes
6. AI Instructions
7. Story Summary

Players can toggle through placeholders with Next/Back buttons and see progress
("Step 1 of 9"). Skipped placeholders become blank in Plot Essentials, Author's Note,
Story Cards, AI Instructions, and Story Summary. In the Opening, skipped placeholders
appear as literal `${text}`.

### Reuse Rules
If the **exact same placeholder text** appears in multiple spots, it's asked once and
filled everywhere. Any difference in spelling, capitalization, or punctuation creates a
separate prompt: `${Character name}` ≠ `${character name}` ≠ `${Character Name}`.

### Special Built-In Placeholders

Seven special placeholders have built-in behavior:

| Placeholder | Behavior |
|------------|----------|
| `${character.name}` | Prompts "Enter your character's name..." |
| `${character.gender}` | Prompts with Male / Female / Custom selection |
| `${character.pronoun.they}` | → he / she / they (based on gender choice) |
| `${character.pronoun.them}` | → him / her / them |
| `${character.pronoun.their}` | → his / her / their |
| `${character.pronoun.theirs}` | → his / hers / theirs |
| `${character.pronoun.themselves}` | → himself / herself / themselves |

The pronoun placeholders auto-fill based on `${character.gender}` — no player input needed.

### Nested Placeholders

You can put a placeholder inside another placeholder. The inner one must be resolved first
(earlier in resolution order), then its answer gets substituted into the outer question:

```
${What is your friend's name?}
${Describe how ${What is your friend's name?} looks}
→ If they enter "Bob", the second question becomes: "Describe how Bob looks"
```

This can get arbitrarily deep. Some creators use nesting to create conversational
registration-style flows where each question references previous answers by name. The
tradeoff: nested placeholders consume a lot of characters, and complex nesting can be
confusing for players.

### Optional/Skippable Placeholders

If a placeholder is the last thing in the Opening, skipping it (pressing Enter with no
input) leaves a blank that the AI will fill in on its own. This creates an "AI decides"
option:

```
The world is threatened by ${What threatens the world? (Leave blank for AI to decide.)}
```

A community trick for toggling optional sections: `${Place a $ here to enable optional
content or leave blank}` followed by `{optional placeholder text}`.

### How Many Is Too Many?

Community consensus and creator experience: **3-5 placeholders is the sweet spot.** 10 is
the practical maximum before players start abandoning. One top creator noted that going
past 6 measurably harms replay rates.

The exception: when the placeholders ARE the experience (like Share A Home's 8+ question
character-sheet wizard, which has 338K plays and the highest save ratio in the top
scenarios). But those questions are each meaningful character-defining choices, not
granular details like hair color or eye color.

Bad: 26 steps asking hair color, eye color, shoe size, favorite food separately.
Good: "Describe your appearance" as a single open-ended placeholder.

**Tip**: include context about what follows the placeholder so players know how to format
their answer: `You have ${Describe yourself. Follows "You have…"}` helps the player write
a fragment that reads naturally in the final text.

### Integration with Story Cards
Player answers can trigger Story Cards. If a player enters "Elena" as their name and a
Story Card has trigger "Elena", that card activates during play.

---

## Publishing and Visibility

### Visibility Levels

| Level       | Who can access          | Searchable? | Use for                      |
|-------------|------------------------|-------------|------------------------------|
| **Private** | Only you               | No          | Work in progress, personal   |
| **Unlisted**| Anyone with direct link | No          | Beta testing, selective sharing |
| **Published** | Everyone             | Yes         | Finished, polished scenarios |

### Content Rating
Scenarios have content ratings: Everyone, Teen, Mature, Unrated. NSFW flag required for
mature content.

### Trending
Published scenarios can appear in Trending. Based on recent play activity — new plays and
engagement boost ranking.

### Description as Marketing
The scenario Description is player-facing only (the AI never sees it). Top-performing
descriptions read like pitches: a hook that sells the fantasy in 2-3 sentences, then
features/credits/update history below the fold. Technical details (scripts used, card
counts, version notes) go at the bottom. Tags should cover genre, theme, and mood —
they're the primary search discovery mechanism.

### Updating Published Scenarios
Changes apply immediately. Existing adventures keep their state; new adventures use the
updated version.

---
