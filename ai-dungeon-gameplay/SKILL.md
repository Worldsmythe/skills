---
name: ai-dungeon-gameplay
description: >
  Play AI Dungeon adventures well and fix them when they drift: manage context during play
  (Plot Essentials, Author's Note, AI Instructions, story cards, Memory Bank), pull cards in
  deliberately by mentioning their trigger keys, use Retry / Edit+Continue and the /reset
  command, steer the narrative with markdown headers and the Do/Say/Story/Continue/See
  action modes, and troubleshoot incoherence. Use when playing or debugging an in-progress
  AI Dungeon adventure. For building a scenario, see the ai-dungeon-scenario-design skill.
---

# AI Dungeon Gameplay

Keeping the AI coherent during play and fixing it when it drifts. For *designing* a scenario
(what to put in each component, story-card format, trigger mechanics), see the
`ai-dungeon-scenario-design` skill; for the platform/API/CLI, the `ai-dungeon` skill.

## How context works

Each turn the engine assembles a context window in this order, and the **beginning and end
get the most model attention**:

```text
1. AI Instructions
2. Plot Essentials
3. World Lore: triggered Story Cards
4. Story Summary
5. Memories
6. Recent Story
7. [Author's note: ...]
8. Last Action
9. Front Memory
10. Buffer Tokens
```

So the strongest live levers are Plot Essentials (near the top), the Author's Note (near the
bottom), and your most recent action. To change behavior *now*, edit the Author's Note or
your last action; to change a standing fact, edit Plot Essentials. Content buried in the
middle (old history, a long card pile) carries the least weight.

**Pull a card in on purpose.** A card enters context only when one of its trigger keys
appears in recent text — and an AI-written trigger fires the card *next* turn, while a
player-typed one fires immediately. If you need a card's info now, name its trigger in your
action ("I think about **Elena**…").

## Tiers and budget

Required elements (AI Instructions, Plot Essentials, Story Summary, Author's Note, Front
Memory, Last Action) can claim ~70% of context. When they compete, the engine keeps Front
Memory and Last Action first, then Author's Note, Plot Essentials, AI Instructions, Story
Summary. Story Cards get squeezed first, so keep their entries compact. Tier numbers change
often — verify live before giving upgrade advice:

| Tier | Context | Memory Bank slots | Monthly Credits |
|------|---------|-------------------|-----------------|
| Wanderer | 4K | 25 | 0 |
| Champion | 8K | 100 | 760 |
| Legend | 16K | 200 | 1,650 |
| Mythic | 32K | 400 | 2,750 |

Shadow tiers above Mythic add more; some models offer Credit-per-action context extensions.
Treat 4K as the default constraint unless the player is on a paid tier.

## Maintaining components during play

The Memory Bank and Auto-Summary handle long-term recall on their own, but they don't
replace curating always-on context:

- **Plot Essentials** is always-on, so it has outsized pull. Update it as facts change
  (injuries, new gear, who's present) and **trim stale lines** — the AI uses everything in
  context, so outdated PE actively steers the story wrong. Name persistent companions here or
  they drift out of the narrative.
- **AI Instructions**: add a rule when a misbehavior *recurs* (modern slang in a medieval
  setting, everyone too agreeable, wrong measurement system). A good starting set rarely
  needs changes.
- **Author's Note**: touch last, and keep it short (it works by position, not length).
  Change it when the scene shifts hard — new setting, tone, or focus.
- **Story Summary**: leave it to Auto-Summary; manual edits feed back into later summaries
  and cause drift.
- **Memory Bank**: just confirm it's on (Gameplay → AI Models → Memory System). It embeds
  roughly every 6 actions and retrieves relevant memories automatically.

**Story cards mid-play:** make one when something worth remembering recurs (a character,
place, item, faction, rule). Keep entries short and factual, and don't restate Plot
Essentials in a card — always-relevant goes in PE, only-when-mentioned goes in a card.

## Steering the narrative

- **Action modes.** Do (`> You ...`), Say (`> You say, "..."`), Story (raw narration — best
  for scene-setting, corrections, and other characters' POV), Continue (extend with no
  input), See (image prompt). A run of Do/Say breeds a repetitive `> You...` rhythm the AI
  mirrors; mix in Story. Start a Do with quoted dialogue or "My…" to skip the auto-prepend.
- **Markdown headers steer hard.** `##` / `###` read as chapter/section breaks from training
  data, so `## The ambush at the bridge` pushes the next beat more forcefully than narrating
  "later, at the bridge" would.
- **Scene breaks.** `---` or `***` alone on a line is a recognized scene break and a single
  token in most tokenizers — cheap and reliable. Pair it with a longer Story action to set
  the new scene and pull the AI off the old one at once.
- **Bring something back** into the AI's attention by putting it in recent text — say it, do
  it, or narrate it. (Naming a card's trigger does this *and* fires the card next turn.)
- **Flashbacks and dreams get baked into memory as if they happened** — the memory system
  can't distinguish them. If you use one, add a PE note like "The events of X were a dream
  and did not actually happen."

## Fixing drift: Retry, /reset, Edit+Continue

- **Edit + Continue** beats Retry for course-correction: edit the bad text to what you want,
  then Continue, so the AI continues from a context you've reshaped.
- **Retry** re-rolls the latest message, stacking alternatives you can swipe through.
- **`/reset`** is typed into the action input — there is no `/reset` button and no `/retry`
  command. It isn't sent as an action (the input clears immediately); it removes the retry
  counter from the previous message, dropping that response's accumulated alternatives. Use
  it to clear a pile of retries before continuing or retrying fresh.

## Common problems and fixes

| Problem | Fix |
|---------|-----|
| AI ignoring Author's Note | Too long. Cut to 2 sentences. Power is positional, not informational. |
| Story Card not firing | Check Context Viewer. Usually a leading-space issue or trigger word not in recent actions. |
| AI repeating "You..." starts | Too many Do/Say actions. Mix in Story mode. |
| AI forgetting NPCs | Make a Story Card. If they're always present, mention in PE instead. |
| Companions disappearing | Name them in PE: "You travel with Bob and Lucy." |
| Negatives not working | "Don't mention X" fails. Say "Avoid X" or state what IS true. |
| Adventure drifting after 50+ actions | Enable Memory System. If already enabled, upgrade context tier. |
| AI continuing a finished scene (16K+) | Add "Current scene: [description]" to PE. Update when scenes change. |
| AI getting facts wrong | Check if the correct info is in PE or a triggered card. If not, add it. |
| AI making up information | Reduce verbose PE/card entries. Dense, factual statements work better than narrative descriptions. |
| Characters acting out of character | Add or update their Story Card with personality constraints. |
