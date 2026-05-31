# AI Dungeon Script Catalog and Compatibility

Which community scripts exist, what each does, how to install and combine them, and which go
inert on Optimized-Context (`cacheEfficient`) models. This is the *selection and install*
guide — for how scripts actually work (hooks, the modifier pattern, state, the sandbox, the
full `cacheEfficient` rule), see the `ai-dungeon-scripting` skill. Installation uses the
`aid` CLI's `scripts` command, which ships with the `ai-dungeon` skill.

## Contents
- [Installing and Combining Scripts](#installing-and-combining-scripts)
- [Crediting Scripts](#crediting-scripts)
- [Script Catalog](#script-catalog)
- [Compatibility at a Glance](#compatibility-at-a-glance)

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
— treat this as "context-text injection won't reach the model." The `ai-dungeon-scripting`
skill's sandbox section has the full mechanism.)

---

## Crediting Scripts

Creators advertise enabled scripts with emoji markers — `[IS🎭]`/`🎭` (Inner Self),
`[🎴]`/`⭐️` (Auto-Cards), `[🦊]` (FoxTweaks) — in the scenario **title and/or description**,
usually with author credit ("Inner Self by LewdLeah"); some also drop `inner self`/`auto
cards` into the tag list. It's a discovery/credit signal on the *scenario* — **not**
something you put on story cards, and not a technical/compatibility requirement. (The emoji
form lives in the title/description, not the tag field.)

---

## Script Catalog

Each entry gives the per-slot chain call (for combining) and notes which feature(s) rely on
rewriting context (and so go inert on `cacheEfficient` models). **Clone from the repo; don't
recreate.**

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

---

## Compatibility at a Glance

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
