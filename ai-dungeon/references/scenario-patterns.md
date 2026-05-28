# AI Dungeon Scenario Patterns & Best Practices

Data-driven patterns from analyzing top community scenarios, plus practical design
guidelines. For structural mechanics (branch trees, scenario types, placeholders), see
`scenario-design.md`.

The card-count and design-pattern breakdowns below are reproducible with the bundled
CLI: `aid analyze popular --deep --sfw --no-official` and `aid analyze trending --deep
--sfw` regenerate them against live data. Use `--no-official` for community analysis —
the platform's own `aidungeon` account authors the default starters and a band of
example scenarios that otherwise dominate the top of `popular` and skew the stats.

## Table of Contents
1. [Patterns from Popular Scenarios](#patterns-from-popular-scenarios)
2. [Design Best Practices](#design-best-practices)
3. [Design Workflow](#design-workflow)

---

## Patterns from Popular Scenarios

Analysis of the top community scenarios (by play count, Everyone+Teen rated, as of mid-2026)
reveals consistent structural patterns.

### Prompt Length: Shorter Than You'd Expect

The most-played scenarios have surprisingly short prompts. The world-building lives in
cards and plot components, not the prompt.

| Scenario (plays) | Prompt approach |
|-------------------|----------------|
| Faerûn (536K, 564 cards) | Single evocative paragraph, pure world-setting |
| 2026 (386K, 145 cards) | Two sentences, open-ended sandbox invitation |
| Share A Home (338K, 0 cards) | Long placeholder chain (8+ `${}` prompts) — the prompt IS the experience |
| MHA (317K, 30 cards) | Meta-prompt: "Choose a premade character or make your own" |
| Isekaied (269K, 53 cards) | Narrative setup → MC choice at end |
| Auto-Cards (258K, 0 cards) | `"Begin."` — literally two words, script-driven |
| Villain's Academy (156K, 89 cards) | `"Choose your preferred context size."` — pure MC navigation |
| Star Wars (152K, 382 cards) | Single paragraph, world-setting |

**Takeaway**: the prompt is a hook, not a world bible. Put lore in cards and Plot Essentials.

### Card Count Is Bimodal (Exclude the Official Account First)

A trap the raw `popular` list sets: the official `aidungeon` account authors the default
starters *and* a stable of example scenarios (Kedar, Xaxas, Penwick, Winterbloom, Planet
Omega, Alarathos, Gorgon, Besatheus), almost all in a tidy 20–35 card band. They're
platform-promoted and sit near the top of `popular`, so the middle of the distribution
looks fuller than the community actually builds. Always analyze with `--no-official`.

Filtering them out (`aid analyze popular --deep --sfw --no-official`, late May 2026) the
community top tier is sharply bimodal: of ~17 scenarios, 7 carry zero cards (placeholder-
or script-driven: Share A Home, Auto-Cards, Supervillain RPG, the SCP Foundation) and 4
are 100+ card bibles (Faerûn 564, All-in-One Fantasy 433, Star Wars 382, 2026 145). The
middle is real but thin and varied — Fiomar 53, Isekaied 53, My Hero Academia 30,
Villain's Academy 89 — not the clean 20–35 band the official examples imply.

**Takeaway**: build a world-bible (100+) or lean on placeholders/scripts (0). A hand-built
mid-size set (30–90) works too, but the half-hearted 5-card scenario is what underperforms.
You either build a world or you let the player/script do the work.

### Multiple Choice Dominates All-Time

5 of the top 8 community scenarios use MC branching. Root prompts are either a narrative
setup ending with "pick your path" (Isekaied) or pure navigation ("Choose your preferred
context size" — Villain's Academy). The branch tree drives replayability and discovery.

Villain's Academy uses a particularly smart pattern: the root branches by **context size**
(low-context for free tier, high-context for paid), so each leaf is optimized for the
player's actual token budget.

### Card Design Patterns (from real scenarios)

**Taxonomy hierarchies** (MHA pattern): a top-level concept card ("Quirks") triggers on
the broad term, then sub-type cards (Emitter, Transformation, Mutant) trigger on both their
specific name AND the parent term. This creates a layered knowledge system where the AI
gets the right level of detail based on what's being discussed.

```
Card: "Quirks" — keys: "Quirks, superhuman ability"
  → defines the concept broadly
Card: "Emitter" — keys: "Emitter, quirks"
  → details the sub-type, also fires when "quirks" is mentioned
Card: "Transformation" — keys: "Transformation, quirks"
  → same pattern
```

**Organizational cards** (MHA, Star Wars): factions, organizations, and key figures get
their own cards with multiple trigger aliases ("The League of Villains, Villain Alliance").

**Practical observations from real card data**:
- Many authors leave Name and Notes empty and put everything in Keys + Entry (Value).
  This works fine — the AI only sees Entry anyway.
- `useForCharacterCreation: true` is set liberally, even on non-character cards like
  quirk-type explanations and faction descriptions. The Character Creator will show all
  cards with this flag, regardless of Type.
- `type: "Custom"` is used far more often than the built-in types (Character, Location,
  etc.) in practice. The Type field is for your own organization, not the AI's.

### Placeholder-Heavy Scenarios Work

Share A Home (338K plays, 16K saves) has zero cards and zero scripting — it's entirely
driven by 8+ placeholder questions that let the player define everything. The key design
trick: **the entire placeholder prompt is duplicated into Plot Essentials** (so the
player's character info persists in always-on context even after the prompt scrolls out
of history). Without this, the AI would eventually forget who the player's characters are.

Your Own Nation (7K plays, 2K saves, Everyone-rated) shows a more structured approach:
Plot Essentials is a labeled template with sections (`[Nation info: ...]`,
`[Player info: ...]`) where 9 placeholders fill in the blanks. The Author's Note combines
a final theme-selection placeholder with hard-coded writing style rules. Zero cards, zero
scripts — the experience is entirely placeholders feeding into well-structured PE.

### How Popular Scenarios Actually Use Plot Components

Examining the full plot component data from top scenarios reveals distinct strategies:

**Card-bible approach** (Faerûn: 536K plays, 564 cards): Empty Plot Essentials, empty
Story Summary, one-sentence Author's Note ("A swashbuckling adventure and cozy fantasy
story set in Faerûn."), detailed AI Instructions (writing style, combat rules, character
voice). The world lives entirely in 564 story cards. PE is empty because everything is
triggered contextually.

**Sandbox approach** (2026: 386K plays, 145 cards): Empty PE, 7-word Author's Note
("Detailed content and description, realistic and modern"), empty AI Instructions. Minimal
steering, maximum freedom, world knowledge in 145 cards.

**Placeholder-template approach** (Your Own Nation: 7K plays, 0 cards): Placeholder-heavy
PE with structured sections, split Author's Note (player theme choice + writing rules),
detailed AI Instructions with behavioral constraints. Zero cards — the player defines
the world through placeholders.

**Placeholder-mirror approach** (Share A Home: 338K plays, 0 cards): Prompt and PE contain
the same placeholder text. The prompt creates the opening; the PE duplicate keeps that info
in always-on context. No AI Instructions, no Author's Note, no cards. Minimal design,
maximum personalization.

**MC navigation approach** (Villain's Academy: 156K plays, 89 cards): Root has completely
empty plot components — all real content lives on leaf branches. The root is pure
navigation ("Choose your preferred context size.").

**Key insight**: the most successful scenarios commit to one strategy rather than spreading
thin across all components. Faerûn puts everything in cards. Share A Home puts everything
in placeholders. 2026 uses cards with minimal steering. Mixing strategies is fine, but
each component should have a clear purpose.

### Script-as-Product Is a Category

Auto-Cards (258K plays) has a prompt that's literally "Begin." and zero cards. The entire
value is the scripting system. The description IS the documentation. This works because
the script (Auto-Cards / Inner Self) provides ongoing value during play, not just at setup.

### Branch-Level Card Isolation (Confirmed by API)

Querying story cards on MC scenario roots returns empty arrays, even when `storyCardCount`
reports 53 or 89 cards. The cards live on the leaf branches. `storyCardCount` on the root
aggregates across the tree, but the actual card data is branch-local. This confirms that
each leaf must carry its own complete card set.

### All-Time vs Trending: Two Different Metas

The all-time popular list and the current trending list reward different designs:

**All-time** is dominated by MC world-bible scenarios (Faerûn, Star Wars, Isekaied,
Villain's Academy) with deep card sets and branching setup flows. These are replayable
systems — players come back to try different branches.

**Trending** (`aid analyze trending --deep --sfw`, top 18 Everyone+Teen) is almost
entirely focused Simple Start scenarios with a single compelling premise, low
card counts, and LewdLeah's scripts (Inner Self, Auto-Cards) handling
persistence and memory. Descriptions read like anime episode synopses or light
novel titles — they sell the fantasy, with technical details (scripts used,
update history) at the bottom.

The meta has shifted: instead of building 500 cards by hand, creators set up one strong
situation and let Auto-Cards generate cards dynamically during play, while Inner Self gives
NPCs persistent goals, memories, and personality tracking.

### Community Script Infrastructure

Two open-source scripts by LewdLeah have become standard infrastructure across the
AI Dungeon creator community:

**Auto-Cards** (`github.com/LewdLeah/Auto-Cards`) — detects named entities during play
and automatically generates/updates story cards. Solves the object-permanence problem
without manual card authoring. Installable into any scenario.

**Inner Self** (`github.com/LewdLeah/Inner-Self`) — gives NPCs persistent inner
monologue, goals, secrets, and self-reflection. Characters remember events and develop
over time. Uses a JSON identity system per character.

As of mid-2026, the majority of trending scenarios include one or both of these scripts.
The `[IS🎭]` and `[🎴]` emoji tags in scenario titles are community shorthand for
"Inner Self enabled" and "Auto-Cards enabled." When helping someone design a scenario,
mentioning these scripts as options is almost always relevant — they're the closest thing
AID has to a standard library.

---

## Design Best Practices

### Prompt (Opening) Writing
- Keep it shorter than you think — a single evocative paragraph often outperforms a wall of text
- The prompt is a hook, not a world bible. Lore goes in cards and Plot Essentials.
- End with something to react to (a question, arriving threat, discovered mystery)
- For MC scenarios, the root prompt just frames the choice — child titles become buttons
- For placeholder-heavy scenarios, the prompt can BE the experience (a wizard-style Q&A)

### Story Card Strategy
- Commit to a real card set or skip cards entirely: 100+ for a world-bible, 30–90 for a
  hand-built mid-size world, or 0 for script-/placeholder-driven scenarios. The
  half-hearted 5-card middle is what underperforms
- Build taxonomy hierarchies: broad concept card + specific sub-type cards sharing a trigger keyword
- Use multiple trigger aliases for key entities ("The League of Villains, Villain Alliance")
- Keep entries to 2-4 sentences; they compete with history for context space
- Cross-reference cards to create chain activation
- Don't bother with the Name/Notes fields unless using Character Creator — the AI only sees Entry
- Test with Context Viewer: play, take turns, check which cards fire

### Plot Component Strategy
- AI Instructions: rules, style, POV, behavioral constraints. Most steering goes here.
  Top scenarios use detailed, specific instructions ("Don't describe thoughts, emotions
  or decisions. Describe what other people say and do." — Faerûn).
- Plot Essentials: only always-relevant facts. Don't duplicate story card content.
  Mentioning key names here primes the AI to use them, which triggers their cards.
  For placeholder scenarios, use PE as the permanent home for player-defined info
  (the "placeholder-mirror" pattern: same placeholders in prompt AND PE, so answers
  persist in always-on context after the prompt scrolls out of history).
- Author's Note: current scene tone, genre cues. Keep short for meta-guidance effect.
  Can be split: one placeholder for player theme choice + hard-coded style instructions.
- Story Summary: usually auto-generated. Only pre-fill for needed backstory.

### Context-Tier-Aware Design
Consider building separate MC branches for different context tiers (Villain's Academy
pattern). A 4K-context free-tier player and a 32K Mythic player have radically different
card budgets. Offering "Low Context" and "High Context" branches with appropriately sized
card sets and Plot Essentials prevents the free-tier experience from being bloated.

### Common Mistakes
1. Giant prompt with no story cards — put stable lore in cards
2. Empty plot components — provide the AI some guidance
3. Untested scripts — broken scripts ruin the experience
4. Content on non-leaf branches — it's invisible to the AI during play (confirmed by API data)
5. Overcomplicated option trees — keep choices manageable
6. Missing key NPC cards — important characters need story cards
7. Cards on MC root nodes — they're ignored during play, put them on leaves

### Quality Assurance
- Playtest from start as a new player, multiple times
- Test every branch/option path
- Verify story cards trigger via Context Viewer
- Check AI behavior matches your intentions
- Have others beta test before publishing

---

## Design Workflow

1. **AI Instructions** — POV, tense, style, behavioral rules
2. **Plot Essentials** — always-true world and character facts
3. **Author's Note** — 3-4 sentences of tone/genre/theme
4. **Story Cards** — cards for major entities (or skip if using Auto-Cards)
5. **Opening (Prompt)** — engaging hook with immediate situation
6. **(Optional) Scenario Options** — branch tree for multiple-choice setup
7. **(Optional) Character Creator** — character-type Story Cards with Notes for player-facing text
8. **(Optional) Placeholders** — `${questions}` for player customization
9. **(Optional) Scripting** — consider Auto-Cards for dynamic world-building, Inner Self
   for NPC persistence, custom scripts for inventories/dice/quests/commands
10. **Test via Context Viewer** — check card activation, budget allocation, AI behavior
