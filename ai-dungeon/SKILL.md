---
name: ai-dungeon
description: >
  Inspect, query, and manage AI Dungeon data through its GraphQL API and the bundled aid
  CLI: auth and tokens, scenario search and details, story-card import/export and conversion,
  trigger-key generation, branch trees, the layered Multiple Choice builder, and the
  platform/context model (Scenario vs Adventure, the context-assembly order, tiers). Use for
  AI Dungeon API, CLI, data inspection, or platform-mechanics questions. For designing
  scenarios see the ai-dungeon-scenario-design skill, for playing see ai-dungeon-gameplay,
  for writing scripts see ai-dungeon-scripting.
---

# AI Dungeon

The platform hub for Phoenix-era AI Dungeon (play.aidungeon.com, beta.aidungeon.com): the
data/API layer and the bundled `aid` CLI. This skill is for *inspecting and managing*
AI Dungeon content; the companion skills cover the craft:

- **ai-dungeon-scenario-design** — designing scenarios (Plot Essentials, Author's Note, AI
  Instructions, story cards, branch trees, placeholders, tags, the script catalog). 
- **ai-dungeon-gameplay** — playing well (context management, Retry/Edit+Continue, `/reset`,
  markdown-header steering, fixing drift).
- **ai-dungeon-scripting** — writing JavaScript scenario scripts (hooks, state, sandbox).

AI Dungeon changes often. For live facts (models, tiers, prices, trending), verify against
the platform rather than trusting these notes.

## Reference Map

- `references/graphql-api.md` — GraphQL endpoint, auth, key queries and mutations, content
  model, search/discovery, draft-vs-published, practical gotchas.
- `references/cli.md` — the `aid` CLI: discovery, details, story-card import/export,
  branch-tree inspection, the `aid mc` layered Multiple Choice builder, offline utilities.
  the CLI's setup/cards JSON, plus the `mc-layers` spec for `aid mc`.

## Core Mental Model

A **Scenario** is a reusable template. An **Adventure** is a single playthrough created from
a scenario. Each turn, AI Dungeon assembles a context window from layered plot components,
triggered story cards, summary/memory systems, recent history, and the latest action, then
sends it to the selected model.

Context-assembly order (the canonical reference; beginning and end positions get the most
model attention):

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

Plot Essentials (near the top) and the Author's Note (near the bottom) are therefore the
highest-leverage steering tools. The `ai-dungeon-gameplay` skill covers using this during
play; the `ai-dungeon-scenario-design` skill covers authoring each component.

A scenario also has a **draft** and a **published snapshot**; publishing is moderation-gated
in the web app. See `references/graphql-api.md` → "Draft vs Published."

## Bundled CLI

The script is `scripts/aid.py`. Examples use `aid` as the installed command name; run it as
`python scripts/aid.py ...` from this skill's folder when it isn't on PATH. Networked
commands need `requests` and a Firebase token; `keys`, `convert`, and `tags` work offline.

```bash
python scripts/aid.py token extract                # how to get a token from the browser
python scripts/aid.py trending --rating everyone --days 7
python scripts/aid.py popular --limit 10
python scripts/aid.py details <shortId>
python scripts/aid.py cards <shortId> --md
python scripts/aid.py tree <shortId>
python scripts/aid.py mc build worlds.spec.json    # compile a layered MC tree (offline)
python scripts/aid.py analyze popular --deep --sfw
python scripts/aid.py keys "elf"
python scripts/aid.py tags fantasy romance darkhumor
```

Full command reference and the `aid mc` spec format: `references/cli.md`.

### Caution with the user's account

The CLI authenticates as the user. Treat public reads very differently from commands that
touch live content: `create`, `duplicate`, `update`, `scripts`, `options`, `card`,
`add-cards`, `import-cards`, `delete`, `restore`, `mc sync`.

- Mutate only with clear, specific permission — but an explicit request to act on the
  user's **own** scenario, with a token they provided or imported, *is* that permission:
  proceed, don't re-litigate the credentials. State the action first, preview when useful,
  and do not reflexively pass `--yes`. Reserve hesitation for destructive ops and
  unprompted/ambiguous actions.
- Be especially careful with `delete`, `import-cards` (replaces the whole card set), and
  `add-cards`/`mc sync` (write multiple branches/cards).
- Never publish on the user's behalf. Publishing is moderation-gated in the web app.
- Do not enumerate libraries, creator catalogs, or broad analysis sweeps unless asked.
  Default to read-only, single-target work.
