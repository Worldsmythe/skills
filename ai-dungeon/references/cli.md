# aid — Bundled AI Dungeon CLI

`scripts/aid.py` is a command-line tool for querying the AI Dungeon GraphQL API and
working with story cards offline. Use it to pull real scenario data (for analysis or
inspiration), inspect how popular scenarios are built, convert story cards between formats,
and generate trigger keys. It pairs directly with the patterns documented in
`scenario-design.md` and `scenario-patterns.md`.

> **Treat the user's account with care.** The CLI authenticates as the user, and the
> commands below the read-only ones (`create`, `duplicate`, `update`, `scripts`,
> `options`, `card`, `add-cards`, `import-cards`, `delete`, `restore`) change their real,
> live content. Run a mutating command only when the user asked for that specific change;
> state what you'll do and don't reflexively pass `--yes`. Be most careful with `delete`
> and `import-cards` (a full card *replacement*). Never publish on their behalf —
> publishing is moderation-gated in the web app, not a CLI action. And don't enumerate or
> scrape: skip `mine`/`creator`/broad `analyze` sweeps unless asked, default to the one
> scenario in front of you, and respect rate limits and the platform's terms.

## Setup

Requires Python 3. Networked commands also need `requests` (`pip install requests`). The
tool talks to the production GraphQL endpoint (`api.aidungeon.com`) and needs a Firebase
auth token for anything that hits the network. Offline commands (`keys`, `convert`, `tags`)
need neither `requests` nor a token.

### Authentication

The API uses Firebase JWTs that expire hourly. The CLI can auto-refresh them if given a
refresh token. To extract tokens, run `aid token extract` for browser-console snippets, then:

```
aid token import '<idToken>' '<refreshToken>'   # enables auto-refresh
aid token status                                 # check expiry
```

Or for a one-off, set `AID_TOKEN='firebase <jwt>'` in the environment, or pass `--token`.

The token store lives at `~/.config/aid-cli/tokens.json`. Tokens auto-refresh when within
5 minutes of expiry.

## Commands

### Discovery (needs token)

| Command | What it does |
|---------|-------------|
| `aid trending [--limit N]` | Trending scenarios this week |
| `aid popular [--limit N]` | All-time popular scenarios |
| `aid search "terms"` | Keyword search |
| `aid mine [--published \| --drafts]` | Your own scenarios, newest first |
| `aid creator <username>` | Another creator's published scenarios, newest first |
| `aid details <shortId>` | Full scenario details + plot components + design pattern label |
| `aid resources` | Your credit/scale balances |

`mine` lists the authenticated user's scenarios (resolving your handle from the token),
sorted by last updated, marking each `published` or `draft`. Default shows both;
`--published` / `--drafts` filter to one side. `creator <username>` does the same for
someone else — published only, since drafts aren't visible to non-owners. Page either
with `--limit`/`--offset`.

`trending`, `popular`, `search`, and `analyze` take creator filters: `--by USER…` keeps
only those creators, `--exclude USER…` drops them, and `--no-official` drops
platform-promoted accounts (the official `aidungeon` account authors the default starters
and a stable of example scenarios that otherwise dominate the top of `popular`). Filtering
is applied to the fetched page, so raise `--limit` if a filter thins the list too much.

The same four commands take rating filters: `--rating everyone teen mature unrated` (any
combination), or the shortcuts `--sfw` (Everyone+Teen) and `--nsfw` (Mature+Unrated).
`--rating`/`--sfw`/`--nsfw` are mutually exclusive; default is all ratings.

`details` now returns the full plot components (Plot Essentials, Author's Note, AI
Instructions, Story Summary) and classifies the scenario's design pattern (card-bible,
placeholder-driven, sandbox, MC-navigation, etc.) using the same taxonomy as
`scenario-patterns.md`.

### Story Cards (needs token)

| Command | What it does |
|---------|-------------|
| `aid cards <shortId>` | List a scenario's story cards, grouped by type |
| `aid cards <shortId> --md` | Export cards in the skill's markdown format |
| `aid cards <shortId> --json` | Raw card JSON |

If a scenario reports a nonzero `storyCardCount` but returns no cards, it's a Multiple
Choice scenario — cards live on leaf branches, not the root. The tool says so and points
you to `aid tree`.

### Creating (needs token)

| Command | What it does |
|---------|-------------|
| `aid create [--title T] [--prompt P] [...]` | Create a new scenario |
| `aid duplicate <shortId>` | Copy a scenario into your library |

`create` takes the same `ScenarioInput` as `update`, so the common fields are
available up front: `--title`, `--description`, `--prompt`, `--plot-essentials`,
`--authors-note`, `--tags`, `--rating`, `--type` (all text flags accept `@file`).
It returns the new shortId; flesh out cards/scripts afterward with `aid update`.

`duplicate` works on any scenario, not just yours — it's how the "Copy of …"
scenarios get made. The copy lands in your library as an unpublished draft.

### Editing (needs token, owner only)

| Command | What it does |
|---------|-------------|
| `aid update <shortId> --description "..."` | Edit scenario metadata/plot |
| `aid update <shortId> ... --dry-run` | Preview the change without sending |
| `aid scripts <shortId> --shared-library @lib.js` | Edit scripts (lightweight) |

`update` is read-modify-write: it fetches the scenario, applies the fields you pass,
and sends the *whole* object back (the API replaces it wholesale, so anything omitted
would be cleared — the tool round-trips everything you didn't touch, including all
story cards and the full scripts blob). It refuses scenarios you don't own.

Editable flags: `--title`, `--description`, `--image`, `--tags` (linted before
sending), `--rating`, `--third-person`/`--no-third-person`,
`--allow-comments`/`--no-allow-comments`, `--scripts-enabled`/`--no-scripts-enabled`,
`--prompt`, `--plot-essentials`, `--authors-note`, `--story-summary`.

Any text flag accepts `@path` to read the value from a file. `--dry-run` prints a
per-field change summary plus payload size and sends nothing (add `--json` to dump the
full mutation input). Pairs with `export`: dump a scenario, edit the files, push back.

**Scripts live in their own command.** `aid scripts <shortId>` edits the four scripts
via a dedicated `updateScenarioScripts` mutation that *merges* — it sends only the
scripts you pass (`--on-input`, `--on-output`, `--on-context`, `--shared-library`, each
accepting `@file`, plus `--scripts-dir DIR` for `input.js`/`output.js`/`context.js`/
`library.js`). Unlike `update`, it doesn't round-trip the whole scenario, so changing a
tiny `onInput` no longer re-uploads a 900KB shared library. `--dry-run` previews.

`--type {simple,multipleChoice,characterCreator}` converts a scenario's type. Note
Multiple Choice and Character Creator scenarios need child option branches to be
playable — create those with `aid options` (below).

### Multiple Choice / Character Creator structure (needs token, owner only)

| Command | What it does |
|---------|-------------|
| `aid options <shortId> [--count N] [--title T]` | Create N child option branches |
| `aid update <childShortId> --prompt ...` | Edit a branch (each is a normal scenario) |
| `aid card <shortId> [fields]` | Create a story card (no `--id`) |
| `aid card <shortId> --id <cardId> [--cc] [fields]` | Edit one story card in place |
| `aid card <shortId> --id <cardId> --delete --yes` | Delete a story card |
| `aid add-cards <shortId> <file>` | Add cards from a file, keeping existing (non-destructive) |
| `aid import-cards <shortId> <file> --yes` | Replace the whole card set from a file (destructive) |

Multiple Choice and Character Creator scenarios are a parent scenario with child
*options*. The parent holds the framing prompt (and, for a Character Creator, the
cards/scripts and the cards flagged `useForCharacterCreation`); each child is a full
scenario with its own shortId. `aid options` adds children (also works on a child
to nest sub-options); list them with `aid tree` (it shows your working draft by
default, so newly created branches appear immediately). Edit any branch with
`aid update <childShortId>` — children take the same fields as any scenario.

`aid card` is a surgical single-card operation via `updateStoryCard`/`deleteStoryCard`,
sending only that one card — far cheaper than the ~1 MB full-scenario `update` when a
scenario carries a big shared library. Three modes, keyed on `--id`:

- **Create** (no `--id`): mints a new card from the fields you pass (`--title`, `--type`,
  `--keys`, `--value`, `--description`, `--cc`). Needs at least `--title` or `--value`;
  keys auto-generate from the title if omitted.
- **Edit** (`--id <cardId>`): round-trips the card and applies the fields you pass. Get
  ids from `aid cards <shortId> --json`.
- **Delete** (`--id <cardId> --delete`): destructive — previews unless you add `--yes`.

There is no separate create-card API; the engine upserts on a client-chosen id. The
create path deliberately avoids reading the scenario's draft state first, because doing so
on a never-published scenario makes the new card land in the published view instead of the
draft (reproducible; mechanism unconfirmed — the read-then-write ordering is what matters).
Text fields accept `@file`.

`aid import-cards <shortId> <file>` replaces the scenario's **entire** card set in one
call (`importStoryCards`) from a `.json` or `.md` file — autodetected, same format as
`convert`/`export`. It's destructive (the old set is discarded), so it previews unless you
pass `--yes`, and like card-create it checks ownership without reading state first to avoid
the draft fork.

`aid add-cards <shortId> <file>` is the **non-destructive** sibling: it keeps existing
cards and adds the file's on top, by upserting each card with a fresh id (one request per
card). Use it to append; use `import-cards` to wholesale-replace. Note the tempting
"export → concat → import" merge does *not* work — reading the current cards to merge them
forks the draft (the merged set lands in published, not the draft), which is exactly why
`add-cards` upserts per-card with no pre-read instead.

`aid delete <shortId>` removes a scenario or option branch you own. It's destructive,
so without `--yes` it only previews what would go; pass `--yes` to actually delete.
Deletes are soft (the API marks `deletedAt`), and `tree`/`export` filter those nodes
out, so a deleted branch stops showing up immediately. Because it's soft,
`aid restore <shortId>` undoes it (clears `deletedAt`).

### Multiple Choice Inspection (needs token)

| Command | What it does |
|---------|-------------|
| `aid tree <shortId>` | Render the MC branch tree (recursive `options`), marking playable leaves |
| `aid tree <shortId> --published` | Same, but the published snapshot instead of your draft |

Walks up to four levels deep and shows card counts per node, so you can see where content
actually lives. Confirms the leaf-vs-non-leaf distinction visually. Like the other
inspection commands, `tree` shows your working draft by default (so in-progress branches
appear); `--published` shows the live snapshot.

### Analysis (needs token)

| Command | What it does |
|---------|-------------|
| `aid analyze popular [--deep]` | Aggregate stats over the popular list |
| `aid analyze trending [--deep]` | Same for trending |
| `aid analyze popular --sfw` | Everyone+Teen ratings only |

Without `--deep`, it aggregates what the search API returns: total/avg/median plays, save
ratios, and tag frequency (with a little histogram). With `--deep`, it fetches per-scenario
details to add card-count distribution and a design-pattern breakdown — this reproduces the
manual analysis used to build this skill, so you can re-run it as the platform evolves.

### Offline Utilities (no token)

| Command | What it does |
|---------|-------------|
| `aid keys "word" [--existing "..."]` | Generate trigger keys via AID's `buildKeys` algorithm |
| `aid convert <file> [--to json\|md] [--out FILE]` | Convert story cards between JSON and markdown |
| `aid tags [tag...]` | Lint tags against AID's rules (no args = show the tag guide) |

**`keys`** ports AID's exact key-building logic: short words (<6 chars) get punctuation
guards on both sides, medium words (6-8) get one-sided guards, long words (9+) are used
bare. Output is capped at the 100-character key limit.

**`convert`** autodetects direction from the input (JSON in → markdown out, and vice versa).
The markdown format is:

````
### Card Title
```json
{ "keys": "...", "type": "location" }
```

The card entry text goes here as the body.

---
````

The json block is optional and holds only non-default metadata (keys that differ from
auto-generated, plus non-default type/description/useForCharacterCreation). When omitted,
keys auto-generate from the title and type defaults to "character". The conversion
round-trips cleanly. For the underlying field meanings (and the `value`-vs-`entry`
naming gotcha), see `gameplay.md` → "Story Cards During Play".

**`tags`** checks the 10-tag limit, flags spaces (engine reads only the first token),
special characters, and uppercase, and emits a cleaned list. With no arguments it prints
the category-based tagging guide.

## Global Flags

- `--token <jwt>` — override stored/env token for one call
- `--json` — raw JSON output (most commands)
- `--limit N` / `--offset N` — pagination for discovery commands
- `--filters 'key=value'` — inject extra search filters as JSON (repeatable)

## Draft vs published

A scenario has two versions: the **working draft** (what the editor shows and what edits
write to) and the **published snapshot** (what the public sees, gated behind a moderation
review). `hasUnpublishedChanges` flags when they diverge. Inspection commands (`details`,
`cards`, `tree`, `export`) default to the **draft** — consistent with the edit commands —
and take `--published` to view the live snapshot. For scenarios you don't own this makes no
difference; the API only ever returns their published version.

One sharp edge: on a never-published scenario, querying the draft state right before a
single-card write makes that card land only in the published view, not the draft —
reproducible, and it doesn't heal over time (mechanism unconfirmed; the read-then-write
ordering is what matters). The `card` create path avoids it by checking ownership without
reading state. Full-scenario `update` and existing-card edits aren't affected.

## Notes

- This hits internal, undocumented endpoints used by the web client. Field names and
  behavior can change. The story card text field is `value` in GraphQL but `entry` in the
  scripting API and UI.
- Search results (`SearchableContent`) don't include `storyCardCount`; only full `Scenario`
  objects do. That's why `analyze --deep` needs per-scenario detail fetches.
- Respect rate limits and the platform's terms. This is a tool for creators studying public
  scenarios and managing their own content, not for bulk scraping.
