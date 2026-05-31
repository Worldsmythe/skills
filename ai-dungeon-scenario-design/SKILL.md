---
name: ai-dungeon-scenario-design
description: >
  Design AI Dungeon scenarios that play well: write surgical Plot Essentials, a short
  Author's Note, focused AI Instructions, and fact-sheet story cards; choose Simple Start,
  Multiple Choice, or Character Creator; use branch trees, placeholders, and tags; and pick
  community scripts from the catalog. Use when creating, structuring, or polishing an AI
  Dungeon scenario. For playing or fixing an adventure, see the ai-dungeon-gameplay skill;
  for writing script code, see the ai-dungeon-scripting skill.
---

# AI Dungeon Scenario Design

## The one rule

The narrative AI writes the story. Your scenario's job is to **inform** that AI, not to
perform for the player. Keep always-on context lean, make every line a checkable fact, and
trust the AI and the player to take it somewhere. Almost every bad scenario is *over*-supplied
context (stacked hooks, flowery cards, restated facts), not under-supplied.

Read the worked example below first — it's the concrete ideal. The rules after it just name
what the example already shows. Mechanics (types, branch trees, placeholders, trigger words,
tags) live in `references/scenario-design.md`; the community script catalog in
`references/script-catalog.md`. The `aid` CLI (in the `ai-dungeon` skill) creates, edits,
and publishes real scenarios.

## A worked example: an ideal Simple Start scenario

A complete, grounded scenario. This is the default shape worth copying: one strong
situation, lean components, a couple of fact-sheet cards, prose only in the opening.

**Title:** The Late Shift

**AI Instructions** (generally left blank to default to model-specific defaults):

```

```

**Plot Essentials** (always-on, so only what's true every single turn):

```
You are ${character.name}, ${character.gender}, ${Describe yourself in 2nd person, i.e. "You have blond hair and blue eyes."}.

You're the night cook at the Stardust Diner off Route 9 — the only place open past midnight for thirty miles. Del, the owner, works the register; he's known you since you were a kid. 
```

**Author's Note** (two lines; steers by position, not length):

```
Perspective: Second Person Present from ${character.name}'s point of view.
Prioritize complex personality over flat statements; let people be tired, evasive, funny.
```

**Opening** (the one place prose belongs; ends on a beat the AI continues from):

```
Rain needles the windows and the coffee's gone bitter on the burner. Past two, and the
booths are empty except for a trucker asleep over his pie. Del works the crossword by the
register, glasses down his nose.

The bell over the door jangles. A woman steps in out of the dark, soaked through, clutching
a paper bag to her chest like it might bolt. She scans the empty room and comes straight for
your counter.
```

**Story Cards** (world reference, fired when named — fact sheets, not mood):

```
### Del
{ "keys": "Del" }
Del owns the Stardust Diner and works the register at night. Sixties, widower, dry. Has
known ${character.name} since ${character.pronoun.they} was a kid and treats ${character.pronoun.them} like family. Won't discuss his late wife, Carol.

### Stardust Diner
{ "keys": "Stardust, diner, Route 9" }
The Stardust Diner stands alone off Route 9, thirty miles from the next open business. Open
24 hours. Chrome and cracked vinyl, half the booths patched with tape. Regulars: long-haul
truckers, insomniacs, the odd cop.
```

**Tags:** `slice of life` `mystery` `dramatic`

### Why it's shaped this way

- **Plot Essentials names the Player and Del and stops.** Premise + the one ever-present person.
  Everything that's only *sometimes* relevant (Del's late wife, the diner's regulars) lives
  in cards, not here. Naming Del in PE also keeps his card primed.
- **The woman in the opening has no card.** She's a hook the AI will develop, not
  established world reference. Don't pre-card things the story hasn't made real yet.
- **One situation, no stacked hooks.** A stranger, a bag, a dead-of-night diner. The player
  decides what it means. No looming war, no second mystery competing for attention.
- **Cards are fact sheets.** Names, relationships, states, a withheld secret — things the
  story could contradict. No "the diner hums with untold stories." Each entry repeats its
  subject's name (the AI sees the entry, never the card title).
- **Prose appears once, in the opening,** and ends mid-moment so the AI's first message
  follows naturally.

## Component rules

**Design order:** AI Instructions (POV, tense, hard rules) → Plot Essentials (the few
always-true facts) → Author's Note (a short perspective/tone line) → Story Cards (entities
that matter only when named) → Opening (the hook the AI writes from) → branches /
placeholders / scripts only if the scenario's shape needs them. Don't fill every box; commit
to one strategy (below) and leave the rest sparse.

**Author's Note — keep it tiny.** Two or three lines. It steers by sitting near the end of
context, not by saying more. Default shape: perspective + a light personality steer, as
above. Add a line only to correct a *specific, observed* misbehavior, stated flatly — e.g.
"Beastkin have human skin with only animal ears and tails; they do not have fur or muzzles.
Great beasts have fur and full animal features." If a line isn't fixing a real problem, cut it.

**Plot Essentials — surgical.** Injected every turn, so every word costs context and pulls
the story. Premise plus the people who are *always* present, short. Anything situational is a
card. Stale or speculative PE actively drags the story off course; trim it as things change.

**Trust the player.** Every hook in always-on context (PE, Author's Note, AI Instructions)
competes for attention at once; stack three and the AI services all of them and lands none —
that's where incoherence comes from. Seed one situation; put optional threads in cards or in
the opening, not in the always-on layer.

**Cards — inform, don't impress.** Looser than PE (they're only in context when triggered)
but still fact sheets for the AI, never prose for the player. The test for a line: *could the
story contradict it?* If not, it's mood — cut it or move tone to the Author's Note. Draft
entries telegraphically:

```
Bad (performs for the reader):
  The Harbor Syndicate's reach is a cold hand around the throat of every soul in Velt...

Good (informs the AI):
  Harbor Syndicate: organized crime running Velt's docks. Led by Maela B. Controls
  smuggling, loan-sharking, dock permits. Hostile to unpaid debtors. Rival: the Canal Watch.
```

A reliable way to get there: write the entry in "caveman" fragments first (noun phrases,
states, no adjectives), then add grammar only if a human must read it — the grammar pass is
where flowery embellishment sneaks back in.

**Opening — write for the AI's first move.** The only place for descriptive prose. Set a
scene and reveal some directions the player might take (as prose, or as a Multiple Choice
tree — both valid). The AI writes a message *immediately* after the opening, so end on a
moment or an arriving beat and let it lead; don't script turn two.

## Antipatterns

- **Flowery / prosaic writing outside the opening.** PE, Author's Note, AI Instructions, and
  cards are fact-and-rule channels. The opening is the only prose.
- **Duplicated information** across components or across cards. Always-true → Plot Essentials
  once; situational → one card. Copies waste context and drift out of sync.
- **Too many always-on hooks** (see Trust the player). One situation, not five.
- **A half-hearted card set.** Either commit to a card-bible (100+) or go card-light and lean
  on the opening/PE (or a script). A token 5-card set is the worst of both.
- **AI writing patterns** like em-dash overuse, rule of threes, and ornamental
  punctuation. Write plainly.

## Proven scenario shapes

Pick one; don't blend by habit.

- **Focused Simple Start** (the worked example, and the modern default): one strong
  situation, lean components, few or zero hand cards. Often paired with a script (Inner Self
  or Auto-Cards) so memory and NPCs persist without a hand-built card-bible.
- **Card-bible:** 100+ cross-referencing cards carry the world; Plot Essentials empty;
  one-line Author's Note; the steering all in AI Instructions. A replayable reference world.
- **Placeholder template / wizard:** the opening (mirrored into Plot Essentials so it
  persists) is a fill-in-the-blanks character/world builder; zero cards. The questions *are*
  the experience.
- **Multiple Choice replay:** a branch tree of premade starts, each leaf self-contained
  (branches don't inherit). Drives replayability. For layered trees, the `aid mc` builder in
  the `ai-dungeon` skill compiles them.

The `ai-dungeon` skill's `assets/` directory has one ready-to-import example per shape.

## Where to go next

- Mechanics — types, branch trees, placeholders, story-card fields, trigger words, tags,
  publishing: `references/scenario-design.md`.
- Community scripts — what exists, compatibility, install/combine: `references/script-catalog.md`.
- Acting on a real scenario (create/edit/publish, card conversion, the `aid mc` layered
  Multiple Choice builder): the `ai-dungeon` skill's `aid` CLI.
- Writing custom script code: the `ai-dungeon-scripting` skill.
