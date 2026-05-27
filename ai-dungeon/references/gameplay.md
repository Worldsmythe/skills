# AI Dungeon Gameplay Reference

How to keep the AI coherent during play, manage plot components mid-adventure, and
troubleshoot common problems. Sourced from Latitude community team guides (Wilmar,
Le Onyx) and experienced creator advice.

## Table of Contents
1. [Managing Plot Components During Play](#managing-plot-components-during-play)
2. [Techniques for Coherence](#techniques-for-coherence)
3. [Common Problems and Fixes](#common-problems-and-fixes)

---

## Managing Plot Components During Play

The Memory Bank helps with long-term coherence, but it's not a substitute for active
plot component management. The more you play, the more you need to curate what the AI
sees.

### Plot Essentials (PE) — Your Always-On Notes

PE is the most important thing to maintain during play.

**What belongs here**:
- Your character description (update it when things change — injuries, new gear, etc.)
- World lore (update regularly to reflect changes)
- Persistent companions: "You are traveling with Bob the knight and Lucy the thief."
  Without this, companions vanish from the narrative.
- On high context (16K+): a "Current scene:" line to prevent the AI from continuing
  a previous scene. Update it when scenes change.

**What doesn't belong**: anything that's only occasionally relevant. That goes in
Story Cards.

**Maintenance**: regularly remove info that's no longer relevant, update facts that have
changed, and trim anything that wasn't mentioned in the story and doesn't need to be.
Information in PE has a big effect on where the story goes — the AI tries to use
everything in context, so stale info actively pulls the story in wrong directions.

### Story Cards (SC) — Occasionally Relevant Info

Create a card whenever something worth remembering appears:
- You meet an interesting character → card
- You discover a cool location → card
- You acquire a notable item → card
- An important scene happens that you want remembered long-term → card

Update cards when things change. Remove info you don't want referenced anymore. The more
you wait, the more cleanup you'll need later.

**Don't duplicate PE content in cards.** If something is always relevant, it belongs in
PE. If it's only relevant when mentioned, it belongs in a card.

### AI Instructions (AIN) — Behavioral Rules

Add new rules as problems appear during play:
- Peasants talking like modern teenagers → add a speech style rule
- Everyone is unrealistically nice → add a line about conflict and antagonism
- AI doesn't mention sounds/smells → add a sensory detail rule
- Characters use the wrong measurement system → add a rule about that

A good starting set of AI Instructions shouldn't need frequent updates, but any new
"rule" you want the AI to follow should be added here.

### Author's Note (AN) — Scene-Level Steering

Change AN when the scene context shifts dramatically:
- Story was on earth, now you're in space → describe the new setting
- Want the AI to focus on politics instead of combat → change the theme
- Tone should shift from lighthearted to dark → update accordingly

Most of the time, managing PE, cards, and AI Instructions is enough. AN should be the
last thing you touch.

### Story Summary

Leave it to Auto Summarization. Don't manually edit it unless you know what you're doing —
manual edits feed back into subsequent auto-summarization and can cause drift.

### Memory Bank

Automated. You don't need to manage it — it creates memories from every ~6 actions,
embeds them as vectors, and retrieves relevant ones each turn. Just make sure it's enabled
(Gameplay → AI Models → Memory System).

---

## Techniques for Coherence

### Make the AI Remember Something

Sometimes you need to bring specific information back into the AI's attention:

**Via dialogue**: "Hey, remember when we found that map in the cave?"
**Via character action** (Do mode): "You think about the warning the old man gave you."
**Via narration** (Story mode): "Of course she knows about the secret passage — she's
the one who discovered it."

These work because they put the relevant information back into recent history, which
the AI weighs heavily.

### Scene Transitions

When you want to move to a new scene, write a longer-than-usual Story action to set it up.
Describe the new location, the significance of what's about to happen, the mood. A
paragraph of scene-setting does two things: it tells the AI what to focus on, AND it makes
the AI stop obsessing over whatever was happening in the previous scene.

Most models understand `---` or `***` on a line by itself as a scene break. These are
single tokens in most tokenizers (Mistral, Llama/Hermes), so they're efficient and
well-recognized:

```
You close the tavern door behind you and step into the cold night air.

---

The next morning, you wake to the sound of bells.
```

### Be Terse in Plot Components

The more different pieces of information the AI has in context, the less efficiently it
uses any single piece. Dense, short entries outperform verbose ones. If a 3-sentence card
entry can be cut to 2 sentences without losing meaning, cut it.

### Avoid Confusing the AI

Flashbacks, dreams, and thought experiments get baked into memories as if they actually
happened. The AI (and the memory system) can't distinguish "this was a dream" from
"this happened" once a memory is created. If you must use these, consider adding a note
in PE: "The events of [X] were a dream and did not actually happen."

### Use Edit + Continue for Course Correction

If the AI goes in a wrong direction, don't just Retry and hope for better luck. **Edit**
the problematic text to what you want, then **Continue**. This is more reliable than
retrying because you've reshaped the context the AI is continuing from.

### Action Mode Mixing

Overusing Do/Say creates a repetitive `> You...` pattern that the AI mirrors. Mix in Story
mode for narrative beats, scene-setting, and other characters' actions. Start a Do with
quoted dialogue or "My..." to bypass the auto-prepending.

---

## Common Problems and Fixes

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
