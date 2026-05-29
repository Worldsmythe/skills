# AI Dungeon Scenario Design Reference

How scenarios are structured, the three scenario types, branch trees, placeholders,
and publishing. For what actually works in practice, see `scenario-patterns.md`.

## Table of Contents
1. [Scenario Structure](#scenario-structure)
2. [Plot Component Conventions](#plot-component-conventions)
3. [Branch Trees: Leaf vs Non-Leaf](#branch-trees)
4. [Scenario Types](#scenario-types)
5. [Placeholders](#placeholders)
6. [Story Card Trigger Words](#story-card-trigger-words)
7. [Publishing and Visibility (incl. Tags)](#publishing-and-visibility)

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

## Plot Component Conventions

How top community scenarios actually *write* each component (sampled from the most-played
Everyone+Teen scenarios; see `scenario-patterns.md` for the strategy-level breakdown). The
components are a division of labor — almost no scenario fills all of them. Choose a home
for each job and leave the rest empty rather than spreading thin.

### Opening prompt

Three archetypes, and length is bimodal — a few words or a full paragraph, rarely between:

- **World-setting paragraph** — one evocative, *third-person* paragraph that establishes
  the setting and stops (no "you," no situation). Used by card-bibles, where the world
  lives in cards: "Land of intrigue, adventure, magic and mysticism, Faerûn is…"
- **Placeholder wizard** — the prompt *is* a setup form in second person:
  "You are ${name}, a ${age} year old ${gender}…". The opening does the character creation.
- **Navigation menu** (MC roots only) — one line that frames the choice ("Choose your
  preferred context size."); the child branch titles are the buttons.
- Occasionally a concrete in-medias-res scene to react to.

Second person is the default for anything player-centered; world-setting paragraphs are
impersonal.

### Plot Essentials

Either empty or a template — rarely free prose.

- **Empty** when story cards carry the world (card-bibles leave PE blank).
- **Labeled / placeholder template** the player or AI fills: `[ Player: - Name: - Gender:
  - Age: - Appearance: … ] [Setting: …]`, or a character sheet of `${}` blanks.
- **Placeholder-mirror** — duplicate the prompt's `${}` answers into PE so they persist in
  always-on context after the opening scrolls out of history (some prefix it with the
  `/remember` script command).
- Otherwise a couple of always-true anchors: "You are a supervillain.", "You are 16,
  enrolled at U.A. High."

Keep it always-true and short; don't restate story-card content.

### Author's Note

Short style/genre tags, or empty — never prose. When used, a compact labeled micro-format:
a genre+tone one-liner ("A swashbuckling adventure and cozy fantasy story set in Faerûn."),
or labeled tags ("GENRE Science Fiction… THEME Jedi, Sith…", "[Writing Style:Fantasy]",
"Writing style: …  Theme: Highschool, Romance"). It steers by position, so keep it a few
lines.

### AI Instructions

The real steering home when a scenario needs control. Recurring ingredients:

- **Role**: "You're a storyteller and gamemaster", "Role: Narrative Writer", "dungeon master".
- **POV / tense**, stated explicitly: "Write in second person, present tense."
- **Behavioral don'ts**: "Don't describe thoughts, emotions or decisions — describe what
  other people say and do."
- **Action-token semantics**: "Text preceded by > is an attempted action"; "unrealistic
  player actions fail."
- **Anti-summarization**: "Continue where the story left off, even mid-sentence."

Custom AI Instructions *replace* the defaults (they don't layer), so restate the basics you
still want.

### Formatting cues that recur

Bracket and label mini-formats appear across all components because the model parses them
reliably: `[Player: - Name: …]`, `GENRE … THEME …`, `## directive`, `[Writing Style:Fantasy]`.
Prefer them over prose paragraphs for structured cues.

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

**Known issues and creator advice** (as of mid-2026):
- Prompt generation frequently gets world details wrong and can produce a weak or generic
  opening. Use Simple Start or MC when the starting point needs to be reliable.
- Character Creator does not automatically preserve player info in Plot Essentials. Tell
  players to copy character details there, or include a temporary PE note that says so.
- The generated prompt appends content after your Opening prompt; this is by design.
- Placeholders (`${}`) are supported in Character Creator mode, despite older guides.
- Quickstart picks a random combination from all fields and can produce nonsense. Warn
  players in the description.
- Character Creator works best for broad world sandboxes with lots of race, class,
  location, and faction content.

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

## Story Card Format

A story card is a small record with these fields:

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

| Field | Purpose | AI-visible? |
|-------|---------|-------------|
| `keys` | Comma-separated trigger words; the entry injects when one appears in recent text | no (controls injection) |
| `value` | The entry text injected into context when triggered | **yes** |
| `type` | Category: `character`, `location`, `faction`, `race`, `class`, `item`, … | no |
| `title` | Display name in the editor | no |
| `description` | Author notes; also player-facing in a Character Creator | no during play |
| `useForCharacterCreation` | Surfaces the card as a pickable option in a Character Creator | n/a |

**Naming gotcha**: the field is `value` in the GraphQL/JSON shape but `entry` in the
scripting API and the web UI — same data, two names. **Repeat the subject's name inside
the `value`**: the AI sees the entry text but not the `title`, so a card titled "Elena"
whose value starts with "She is…" gives the AI no anchor.

**What belongs in a card — world reference, not plot.** Card entries describe *what
exists*: people, places, factions, races, classes, items, lore, recurring events — surfaced
only when their keys are mentioned. In real card-bibles the entries are uniformly
encyclopedic (Faerûn's `drow` race and the `Bregan D'aerthe` mercenary company; Star Wars
planets; Fiomar's classes/races). They are **not** adventure hooks, quest setups, or premise
("an unexpected encounter at work", "late-night spirals") — that belongs in the prompt and
Plot Essentials. Quick test: if it reads like a wiki entry that fires on a keyword, it's a
card; if it's a situation you want the story to start from or move toward, it isn't.

For the full GraphQL type (including `updatedAt`, `deletedAt`, `factionName`) see
`graphql-api.md` → "StoryCard Object". For the markdown authoring format and JSON↔markdown
conversion, see `cli.md` → the `convert` command.

## Story Card Trigger Words

Trigger words decide when a card's entry gets injected. Getting them right is the
difference between a card that fires reliably and one that never fires (or fires
constantly on the wrong words). The core complication is **spaces**.

### How Spaces Affect Matching

A trigger is matched as a substring of recent text. Spaces around the trigger control
how greedy that match is. The same word has four meaningfully different forms:

| Trigger form | Matches | Risk |
|--------------|---------|------|
| `elf` (no spaces) | "elf", "shelf", "self", "elves"... | Bleeds into longer words |
| ` elf` (leading space) | " elf" but not "shelf" | Won't fire at start of line/dialogue |
| `elf ` (trailing space) | "elf " but not "elfin" | Won't fire before punctuation |
| ` elf ` (both spaces) | only " elf " | Most precise, most fragile |

**The bleeding problem**: a no-space trigger like `elf` fires on "shelf", "self", and
"yourself" because the letters are contained inside those words. For short or common
substrings, add spaces to prevent false positives.

**The start-of-line problem**: a leading-space trigger like ` elf` won't fire when "elf"
starts a sentence or comes right after a quote mark (`"elf"`), because there's no space
preceding it. Cover the common preceding characters with extra triggers: ` elf,"elf,'elf`.

### Rules and Shortcuts

- **Capitalization doesn't matter.** `elf` and `ELF` are equivalent.
- **Plural "s" is automatic** — `frisbee` also catches `frisbees`, *unless* you added a
  trailing space (`frisbee `). Irregular plurals are NOT automatic — `elf` will not catch
  `elves`; add a separate trigger.
- **Longer phrases protect against bleeding.** A `dragon` trigger fires inside "Yellow
  Dragon Inn". But if the dragon's trigger is `bronze dragon`, the Inn won't trigger it
  (it lacks "bronze").
- **Hyphens/symbols separate words.** `Yellow-Dragon` is treated as one token, so a
  ` Dragon` trigger (with leading space) won't fire on it.
- **AI-written triggers fire next turn.** If the AI's output contains a trigger word, the
  card activates on the *following* action, not the current one. Player-typed triggers fire
  immediately.

### Stubbing: The Pro Move

Stub words to a common, specific root instead of listing every variant. To catch all of
"therapy, therapies, therapist, therapeutic", use a single trigger: `therap`. It's specific
enough not to collide with anything unintended, and covers the whole family. This consolidates
triggers and saves space in the 100-char key budget.

### Generating Triggers

The `keys` command in the bundled CLI (`scripts/aid.py`) implements AutoCards'
`buildKeys` algorithm — give it a word and it produces a properly space-and-punctuation-guarded
trigger set sized to the word's length (short words get both-sided guards, long words are
used bare). See `references/cli.md` for more details.

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

### Tags

Tags are the primary discovery mechanism. AID's engine has strict formatting rules:

- **Lowercase.** Tags are case-insensitive; `Romance` and `romance` are the same.
- **Alphanumeric only.** No punctuation, emojis, slashes. `scifi` is more conventional than `sci-fi`.
- **10 tags maximum.** It's a hard limit, not a suggestion. Be strategic.

Suggested category budget (from Yuki's community tagging guide):
- **Genre** (max 2): fantasy, scifi, romance, horror, isekai, mystery, comedy, sliceoflife
- **Themes** (as needed): redemption, betrayal, coming of age, found family, revenge, power fantasy
- **Setting** (max 3): medieval, modern, futuristic, dystopia, space, academy, virtual reality
- **Tone** (1-3): comedic, dramatic, dark, wholesome, mysterious, edgy
- **Style**: second person, sandbox, quest-based, nonlinear, scripted
- **Custom features**: dicerolls, inventory, companion, questlog, skill checks, player choice

The `tags` command in the bundled CLI lints a tag list against these rules and auto-fixes
spaces, casing, and special characters.

### Trending
Published scenarios can appear in Trending. Based on recent play activity — new plays and
engagement boost ranking.

### Description as Marketing
The scenario Description is player-facing only; the AI never sees it. Treat it as a short
pitch first, then put features, credits, scripts used, card counts, and version notes below
the fold. Tags cover genre, theme, and mood for search.

**Script-enabled markers.** Creators advertise enabled scripts with emoji markers — `[IS🎭]`
(Inner Self), `[🎴]` (Auto-Cards), `[🦊]` (FoxTweaks) — in the **title and/or description**,
typically with author credit ("Inner Self by LewdLeah"); some also add `inner self`/`auto
cards` to the tag list. They're a discovery/credit signal, **not** something that goes on
story cards and not a technical requirement. (Tags themselves are alphanumeric-only, so the
emoji form lives in the title/description, not the tag field.)

### Updating Published Scenarios
Changes apply immediately. Existing adventures keep their state; new adventures use the
updated version.

---
