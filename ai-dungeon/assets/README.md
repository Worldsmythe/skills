# Example Scenario Assets

One ready-to-import example per design **shape** from the `ai-dungeon-scenario-design` skill,
in the exact JSON the bundled CLI reads and writes. They're teaching artifacts: hand-authored
from scratch, not copies of any creator's scenario. `focused-start/` is the canonical ideal
(the "Late Shift" worked example from that skill); the rest demonstrate the other valid
shapes and a couple of type/technique mechanics. Most of the card-based examples share one
original setting, the storm-city of Saltspire, so you can see cards cross-reference each
other.

## File formats

Two file types match `aid export` output and round-trip through the CLI; a third feeds the
`mc` tree builder:

- **`*.setup.json`** — a scenario's plot components and metadata, as a flat object:
  `title, description, type, tags[], thirdPerson, nsfw, contentRating, prompt,
  plotEssentials, authorsNote, aiInstructions, storySummary`. The `type` is one of
  `simple`, `characterCreator`, or `multipleChoice`.
- **`*.cards.json`** — a story-card array, each card:
  `keys, value, type, title, description, useForCharacterCreation`. The AI only ever sees
  `value` (and `keys` decide when it fires); `title`/`description`/`type` are for the author.
- **`*.spec.json`** — a *layer spec* for `aid mc build`/`aid mc sync`: a `layers[]` array of
  choice levels (each with `options[]` carrying per-option `plotEssentials`/`cards`/…) plus a
  `leaf` base. The builder compiles it into a full Multiple Choice tree, baking each
  root→leaf path's accumulated content into the leaf. See `../references/cli.md` → "Build a
  Multiple Choice tree from one file".

To use one:

```
aid convert focused-start/cards.json --to md          # read cards as markdown
aid create --title "My City" --prompt @card-bible/setup.json   # (text flags take @file)
aid import-cards <yourShortId> focused-start/cards.json --yes  # load a card set
```

(`setup.json` is a reference template — `aid create`/`update` take the individual fields as
flags, several of which accept `@file`. Story-card files load directly via
`import-cards`/`add-cards`/`convert`.)

## The examples

| Folder | Shape it demonstrates |
|--------|------------------------|
| `focused-start/` | **Focused Simple Start — the canonical ideal.** One situation, lean components, blank AI Instructions, perspective-format Author's Note, two terse fact-sheet cards, `${character.*}` placeholders |
| `card-bible/` | Card-bible: the world lives in cross-referencing cards; empty PE, tone-only Author's Note |
| `placeholder/mirror.setup.json` | Placeholder template: a `${}` prompt mirrored into Plot Essentials so answers persist; 0 cards |
| `placeholder/wizard.setup.json` | Placeholder wizard: one long fill-in-the-blank opening builds the character; 0 cards |
| `mc-context-tier/` | Multiple Choice replay: a root branches by context size; each self-contained leaf carries a right-sized card budget |
| `mc-layers/worlds.spec.json` | Layer spec for the `aid mc` builder: setting → species → era, compiled into a full tree (not a scenario export) |
| `mc-flags/world.spec.json` | Layer spec using **flags + `when` conditions**: context/race set flags that the age/start layers consume to gate lore cards and detail (not a scenario export) |
| `sandbox/` | Minimal steering: one-line hook, short Author's Note, empty AI Instructions; world in a few cards |
| `taxonomy-cards/` | Technique + Character Creator type: a concept card plus subtype cards sharing a trigger, so detail cascades |
| `script-driven/setup.json` | Script-as-product: a `"Begin."` opening and 0 cards — the value is an installed script |

Provenance for the modeled-on scenarios (for study, not imitation): card-bible ← Faerûn
(`3qWxqEvpEHnF`); taxonomy-cards ← My Hero Academia (`dxCHBboSJBlA`); placeholder/mirror ←
Share A Home (`lkka9mZty2mf`); placeholder/wizard ← Supervillain RPG (`2iTOxixO`); sandbox ←
2026 (`0rKJ_eSjEMc6`); mc-context-tier ← Villain's Academy (`W0K-pqKcRkxq`); script-driven ←
Auto-Cards (`Ddt0Akd-lVtj`).

## What to study in each

- **focused-start** — the canonical shape. Plot Essentials names the player (via
  `${character.*}`) and Del, then stops; everything situational (Del's late wife, the
  regulars) is a card or left for the AI. AI Instructions are blank (model defaults are
  usually fine). The Author's Note is perspective + one steer. The stranger in the opening
  has *no* card — she's a hook for the AI to develop, not established world reference. Prose
  appears only in the opening, ending mid-moment so the AI's first message follows.

- **card-bible** — Eight cards (locations, factions, lore, characters) that name and
  reference each other (Saltspire → Lantern Spire/Underkeel; Lantern Guild → Veyra; Pact →
  Coromb), so triggering one primes the next. Plot Essentials is empty: in a card-bible,
  knowledge is contextual, not always-on.

- **placeholder/mirror** — The opening and Plot Essentials hold the *same* `${}` text. The
  prompt creates the scene; the PE copy keeps the player's answers in always-on context after
  the opening scrolls out of history. Without the mirror, the AI forgets the companions.

- **placeholder/wizard** — No mirror, no cards: the experience is the chain of build
  questions. Good when the player defines a character once and plays forward.

- **mc-context-tier** — `_root.setup.json` is `multipleChoice` with empty plot components,
  because only leaf branches affect play. The two leaves are the same opening at different
  card budgets: `high-context` carries the full 8-card set; `low-context` trims to 3 and moves
  the world summary into Plot Essentials. Same scenario, right-sized per tier.

- **mc-layers** — A *layer spec* for the `mc` builder, not a scenario export. Three layers
  (setting / species / era), two options each → 8 self-contained leaves. Run
  `aid mc build mc-layers/worlds.spec.json` to see the tree, then `--out DIR` to inspect a
  baked leaf (e.g. Cyberpunk · Synthetic · Age of Collapse carries all three PE fragments and
  both path cards, because branches don't inherit).

- **mc-flags** — The flag system on top of the same builder. Four layers (context / race /
  age / start) → 16 leaves, with `world.spec.json` setting flags and `lore.json` gating cards
  by `when`. Run `aid mc build mc-flags/world.spec.json` and read the per-leaf flag readout:
  the same `The Sundering` card has a terse and a detailed variant chosen by `context`, and
  `Long Memory`/`Wardstones` appear only when the right race/age/context flags line up. The
  pattern: early layers set flags that don't add content, later layers consume them.

- **sandbox** — Minimal design: trust the model, put world facts in a few cards, steer almost
  not at all. The opposite end from card-bible's density.

- **taxonomy-cards** — Doubles as the Character Creator type example. The `Embers` concept
  card defines the system; each subtype card (`Forge`, `Bulwark`, `Veil`) fires on **both its
  own form (`Forge-kindled`) and the shared term `Kindling`**, so a mention gets the right
  depth. Triggers are guarded against substring bleed (`Kindling`, not bare `Ember`/`Forge`,
  which would catch `remember`/`forged`); run `aid keys` on a short trigger to see why.

- **script-driven** — Deliberately almost empty: the value comes from an installed script
  (Auto-Cards, Inner Self). **Scripts are not bundled.** Install by cloning the canonical
  repo, never by recreating — see the `ai-dungeon-scenario-design` skill's script catalog
  (and the `ai-dungeon-scripting` skill for how the hooks work).
