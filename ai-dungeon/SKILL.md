---
name: ai-dungeon
description: >
  Reference guide for AI Dungeon gameplay, scenario design, story cards, plot
  components, context budgeting, tier limits, models, GraphQL API usage, and
  scripting. Use when the user mentions AI Dungeon, AID, story cards, author's
  note in an AI Dungeon context, Plot Essentials, Do/Say/Story/Continue actions,
  scenario branches, Character Creator, AI Dungeon adventures, AI Dungeon scripts,
  or wants to brainstorm, design, debug, analyze, or publish an AI Dungeon scenario.
---

# AI Dungeon

Use this skill for Phoenix-era AI Dungeon work on play.aidungeon.com and
beta.aidungeon.com: gameplay coaching, scenario design, story cards, plot
components, scripting, API inspection, and context-budget reasoning.

## Working Process

1. Identify the task shape: design/brainstorming, gameplay troubleshooting,
   scripting, API/data inspection, or publishing polish.
2. Read only the relevant reference files below. Do not load every reference by
   default.
3. For current platform facts such as models, tiers, trending scenarios, prices,
   or limits, verify live when possible. AI Dungeon changes often.
4. Give concrete artifacts when useful: Plot Essentials text, AI Instructions,
   Author's Note, opening prompt, story cards, branch outline, script snippets,
   or tag lists.
5. For scripts and generated scenario content, keep the first version practical
   and testable. Prefer compact, direct components over world-bible prose.

## Reference Map

- `references/scenario-design.md`: scenario structure, branch trees, leaf vs
  non-leaf behavior, scenario types, placeholders, trigger words, tags, publishing
- `references/scenario-patterns.md`: popular-scenario patterns, component strategy,
  card strategy, design workflow, community script infrastructure
- `references/gameplay.md`: context and tier budgeting, component maintenance,
  story card usage during play, coherence techniques, troubleshooting
- `references/scripting.md`: JavaScript hooks, state, memory overrides, story card
  APIs, sandbox limits, script patterns, TypeScript declarations
- `references/graphql-api.md`: GraphQL endpoint, auth, key queries, content model,
  search/discovery notes

## Core Mental Model

A Scenario is a reusable template. An Adventure is a single playthrough created
from a scenario. Each turn, AI Dungeon assembles context from layered plot
components, triggered story cards, summary/memory systems, recent history, and
the latest action, then sends that context to the selected model.

Context order:

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

Beginning and end positions are strongest. Plot Essentials and Author's Note are
therefore the highest-leverage steering tools.

## Design Defaults

Start scenario design in this order unless the user's request points elsewhere:

1. AI Instructions: POV, tense, style, behavior rules, hard constraints
2. Plot Essentials: always-true facts about protagonist, world, companions, goals
3. Author's Note: short tone/genre/current-scene steering
4. Story Cards: entities that matter only when triggered
5. Opening: short hook with an immediate situation or choice
6. Branches, placeholders, Character Creator, or scripts only when they serve the
   scenario's actual shape

Do not spread a scenario thin across every system by habit. Successful designs
usually commit to one primary strategy: card bible, placeholder template,
script-driven play, focused Simple Start premise, or Multiple Choice replay
structure.

## Critical AI Dungeon Gotchas

- Multiple Choice non-leaf branches are menu/setup only. Their plot components,
  story cards, scripts, and placeholders do not enter the final adventure.
- Child branches do not inherit parent content. Each playable leaf needs its own
  complete plot components and cards.
- Story Card Name, Type, Triggers, and Notes are not AI-visible during play.
  The Entry is what gets injected.
- Repeat the subject name inside every Story Card Entry.
- Trigger matching is case-insensitive substring matching. Short/common triggers
  need space or punctuation guards to avoid false positives.
- AI-written trigger words usually activate cards on the next turn, not the
  current one.
- Custom AI Instructions replace the defaults; they do not layer on top.
- Author's Note works because of position, not length. Keep it short.
- Story Summary and Memory Bank are automated persistence systems, but stale Plot
  Essentials and verbose cards can still pull the story off course.
- Edit + Continue is usually a better correction loop than repeated retries.

## Action Modes

- Do: character action, shown to the model as `> You ...`
- Say: dialogue, shown as `> You say, "..."`
- Story: raw narration, best for scene setting, corrections, and non-player POVs
- Continue: no new input; the model extends the latest story state
- See: image generation prompt

Overusing Do/Say creates repetitive `> You...` rhythm. Mix Story mode for
scene-setting and correction.
