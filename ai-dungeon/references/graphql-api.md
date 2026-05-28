# AI Dungeon GraphQL API Reference

AI Dungeon's web client communicates with a GraphQL API. This documents the API surface
as observed from the Phoenix client. **Not an official public API** — internal endpoints
used by the web client, subject to change.

## Table of Contents
1. [Endpoint and Auth](#endpoint-and-auth)
2. [Key Queries](#key-queries)
3. [Content Model](#content-model)
4. [Search and Discovery](#search-and-discovery)
5. [Mutations](#mutations)
6. [Draft vs Published](#draft-vs-published)
7. [Practical Gotchas](#practical-gotchas)

---

## Endpoint and Auth

```
POST https://api.aidungeon.com/graphql          (production: play.aidungeon.com)
POST https://api-beta.aidungeon.com/graphql     (beta: beta.aidungeon.com)
POST https://api-alpha.aidungeon.com/graphql    (alpha: alpha.aidungeon.com)
WebSocket: wss://api[-env].aidungeon.com/graphql (graphql-transport-ws)
```

Requests are batched as JSON arrays (multiple operations per request).

### Headers
```
Authorization: firebase <Firebase JWT>
Content-Type: application/json
x-client-version: <version>
x-gql-operation-name: <operation name>
```

Firebase auth uses project `aidungeon-2c6cc`. The API has **read-after-write
inconsistency**: a query immediately after a mutation may return stale data.

---

## Key Queries

### GetScenarioDetails

Fetches full scenario metadata including the opening prompt.

```graphql
query GetScenarioDetails($shortId: String, $viewPublished: Boolean) {
  scenario(shortId: $shortId, viewPublished: $viewPublished) {
    id shortId title description advancedDescription image
    isOwner published tags adventuresPlayed thirdPerson
    nsfw contentRating voteCount saveCount storyCardCount commentCount
    state(viewPublished: $viewPublished) { prompt }
    user { id username isMember profile { title thumbImageUrl } }
  }
}
```

### GetRecentlyPlayed

Returns the user's recently played adventures and scenarios.

```graphql
query GetRecentlyPlayed($interval: String, $limit: Int) {
  recentlyPlayed(interval: $interval, limit: $limit) {
    id deletedAt isOwner
    ... on Adventure { userJoined }
    ...CardSearchable
  }
}
```

Variables: `{ interval: "all", limit: 10 }`

### OpenSearch (Discover)

Powers the Discover page. Supports trending, popular, and filtered searches.

```graphql
query OpenSearch($input: SearchInput!) {
  search(input: $input) {
    items { ...SearchResultFields }
    total hasMore took
  }
}
```

#### Trending (last 7 days)
```json
{ "input": {
    "contentType": ["scenario"], "sortOrder": "trending",
    "contentRatingFilters": ["Mature","Unrated","Teen","Everyone"],
    "contentFilters": ["published"], "screen": "discover",
    "timeRange": "7", "limit": 30, "offset": 0
}}
```

#### Popular (all time)
Same as above but `"sortOrder": "popular"` and omit `timeRange`.

Pagination: increment `offset` by `limit` (0, 30, 60...). Response includes `hasMore`.

### GetResources

```graphql
query GetResources {
  user { id resources {
    creditsBalance { currentBalance }
    scalesBalance { currentBalance }
    promoActionsBalance { currentBalance }
  }}
}
```

---

## Content Model

### Searchable (shared interface)

Both Scenarios and Adventures implement `Searchable`:

```graphql
fragment CardSearchable on Searchable {
  id contentType publicId shortId title description image tags
  voteCount published unlisted publishedAt createdAt editedAt
  deletedAt blockedAt saveCount commentCount userId contentRating
  user { id username isMember profile { id title thumbImageUrl } }
}
```

### Adventure-specific
```graphql
... on Adventure { actionCount userJoined playerCount unlisted
  contentResponses { userVote isSaved isDisliked isCommentFollowed } }
```

### Scenario-specific
```graphql
... on Scenario { adventuresPlayed hasUnpublishedChanges storyCardCount
  thirdPerson nsfw
  storyCards { id title type keys value description useForCharacterCreation }
  contentResponses { userVote isSaved isDisliked isCommentFollowed } }
```

### StoryCard Object

**Field name gotcha**: the GraphQL schema uses `value` for the card's AI-facing text,
but the scripting API, help docs, and UI all call this field `entry`. Same data, different names.

```graphql
type StoryCard {
  id: String!
  updatedAt: DateTime!
  deletedAt: DateTime
  keys: String           # comma-separated trigger keywords
  value: String          # AI-facing text (called "entry" in scripting/UI)
  type: String           # category: Character, Location, Custom, etc.
  title: String          # display name (AI cannot see this)
  description: String    # notes (AI cannot see this; player-facing in Character Creator)
  useForCharacterCreation: Boolean
  userId: String
  factionName: String
}
```

**Note on MC scenarios**: querying `storyCards` on a Multiple Choice root returns an empty
array even when `storyCardCount` is nonzero. The count aggregates across the branch tree,
but cards live on individual branches. Query leaf branches directly for their cards.

### AI Instructions Internal Structure

Stored as `{ type: "scenario", scenario: "text" }` when filled, or `{}` when empty.
The UI flattens this to a plain string.

### SearchableContent (extended)

OpenSearch results include additional fields: `interestFacetId`, `curationTags`,
`heroesVersion`, `publishedVersion`, `qualityRatingSummary` (totalVoteCount,
positiveRatingPercent), and character/world fields for Heroes/Voyage integration
(slotNumber, characterName, className, raceName, worldTitle, etc.).

---

## Search and Discovery

### Sort Orders
- `trending` — weighted by recent engagement, requires `timeRange`
- `popular` — by total engagement, optional `timeRange` (omit for all-time)

### Filters
- `contentType`: `["scenario"]`, `["adventure"]`, or both
- `contentRatingFilters`: `["Everyone", "Teen", "Mature", "Unrated"]`
- `contentFilters`: `["published"]`
- `thirdPerson`: boolean
- `safe`: boolean
- `timeRange`: `"7"` (week), `"30"` (month), or omit for all-time

### Pagination
Offset-based: `limit` (typically 30) + `offset` (0, 30, 60...).
Response: `{ items, total, hasMore, took }`.

### Profile listing (a creator's scenarios)

Same `search` field, with `screen: "profile"` + `username` and `sortOrder: "updated"`:

```json
{ "input": {
    "contentType": ["scenario"], "sortOrder": "updated", "screen": "profile",
    "username": "Worldsmythe", "contentRatingFilters": ["Mature","Unrated","Teen","Everyone"],
    "limit": 30, "offset": 0
}}
```

- **Your own profile**: `published: true` / `published: false` filters to one side;
  omit the key for both (drafts included, because you own them).
- **Someone else's profile**: only their published scenarios are visible — use
  `contentFilters: ["published"]` (their drafts never return).
- **`total` is the returned page size here, not the grand total** — it reads ~5 at
  `limit:5` and ~44 at `limit:50` for the same profile. Rely on `hasMore` and paginate;
  don't treat `total` as a count.

The official `aidungeon` account authors the default starters (Original Quickstart has
56M+ plays) and a band of 20–35 card example scenarios that dominate the top of `popular`.
There's no server-side creator exclusion; filter client-side on `user.username`.

---

## Mutations

All take their input under `variables.input` unless noted. `__typename` keys are tolerated
in input objects but not required — strip them.

### updateScenario(input: ScenarioInput)

**Whole-object replace.** The server overwrites the scenario with exactly what you send, so
any field you omit is cleared. Editing one field means read-modify-write: fetch the full
state (see `GetScenarioState` below), change the one field, send everything back —
including every story card and the entire `scripts` blob.

```graphql
mutation UpdateScenario($input: ScenarioInput) {
  updateScenario(input: $input) { scenario { id shortId title editedAt } message success }
}
```

`input` (the shape the web client sends): `shortId`, `title`, `description`, `tags`,
`contentRating`, `thirdPerson`, `allowComments`, `image`, `type`, `scriptsEnabled`, and a
nested `details` object = the scenario state: `scenarioId`, `type` (always `"scenario"`),
`prompt`, `authorsNote`, `plotEssentials`, `storyCards[]`, `instructions` (JSON, `{}` when
empty), `storySummary`, `storyCardInstructions`, `storyCardStoryInformation`,
`scenarioStateVersion`, `scripts { onInput onOutput onModelContext sharedLibrary }`.

- Top-level **`type`** converts the scenario: `simple` / `multipleChoice` /
  `characterCreator`. MC and CC also need child options (below) to be playable.
- **`scenarioStateVersion` is passthrough** — send the value you read. It is *not* an
  optimistic-concurrency gate and does not increment on update.
- Works on any node you own, including MC/CC child branches (each child is a full scenario
  addressed by its own `shortId`).

The read side is the `ScenarioState` fragment (the editor's `GetScenarioState`), which is
the only query that returns `scripts` and `scenarioStateVersion`:

```graphql
fragment ScenarioState on Scenario {
  state(viewPublished: false) {
    scenarioId type prompt authorsNote plotEssentials
    storyCards { id updatedAt keys value type title description useForCharacterCreation }
    instructions storySummary storyCardInstructions storyCardStoryInformation
    scenarioStateVersion
    scripts { onInput onOutput onModelContext sharedLibrary }
  }
}
```

### createScenario(input: ScenarioInput)

Same input type as `updateScenario`; minimal create is `{ title, details: { prompt } }`.
Returns the new `shortId`. Build the skeleton, then flesh out via `updateScenario`.

### duplicateScenario(shortId: String)

Copies a scenario into the caller's library as an unpublished draft. Works on scenarios you
**don't** own — this is how "Copy of …" scenarios are made.

### deleteScenario(shortId: String)

**Soft delete** — sets `deletedAt` but the node keeps being returned by `options` queries.
Filter `deletedAt != null` client-side or deleted branches linger in a tree.

### createScenarioOptions(title: String, shortId: String, count: Int)

Creates `count` child option scenarios under `shortId`. Used both to give an MC/CC parent
its branches and to nest sub-options under an existing child. Each returned scenario is a
full scenario with its own `shortId`, edited via `updateScenario`.

### updateStoryCard(input: UpdateStoryCardInput!)

`{ id, shortId, contentType: "scenario", type, title, description, keys, value,
useForCharacterCreation }`. **Also the create path for single cards.** A `createStoryCard`
mutation exists but the editor's manual "add card" doesn't use it — it mints a client id
and upserts here (the server honors a never-seen `id`). Far cheaper than a full
`updateScenario` when the scenario carries a large `sharedLibrary`. See the
read-before-write gotcha below.

### deleteStoryCard(input: DeleteStoryCardInput!)

`{ id, shortId, contentType: "scenario" }`. Soft delete (sets the card's `deletedAt`).

### updateScenarioScripts(shortId: String, gameCode: JSONObject)

Scripts-only edit. `gameCode` is `{ onInput, onOutput, onModelContext, sharedLibrary }`
and **merges** — send only the scripts you're changing and the rest are preserved
(verified). Avoids round-tripping the whole scenario (or a multi-hundred-KB shared library
you aren't touching) just to tweak one hook.

### importStoryCards(input: ImportStoryCardsInput!)

`{ shortId, contentType: "scenario", storyCards: [{ keys, value, type, title, description,
useForCharacterCreation }] }` — cards take no `id` (the server mints them). **Destructive:
replaces the scenario's entire card set, not an append** (verified). Subject to the same
read-before-write fork as single-card writes, so don't read `state(viewPublished:false)`
right before it. There is no non-destructive bulk import: a read-merge-import "append"
forks (the merged set lands in published, not the draft) even after a full `updateScenario`
save — to add cards without discarding, loop `updateStoryCard` upserts (fresh ids, no
pre-read) instead.

### restoreScenario(shortId) / destroyScenario(shortId)

`deleteScenario` is a soft delete (`deletedAt`); `restoreScenario` undoes it, and
`destroyScenario` is the permanent hard delete.

---

## Draft vs Published

Every scenario has two versions: a **working draft** (what the editor shows and what
mutations write to) and a **published snapshot** (what the public sees, gated behind a
moderation review). `viewPublished` (on `scenario(...)` and `state(...)`) selects which.

| Field | Meaning |
|-------|---------|
| `published` | Has a public version ever existed? |
| `hasUnpublishedChanges` | Does the working draft differ from the published snapshot? |
| `isPublishedSnapshot` | Echoes the `viewPublished` you requested — a property of the *response*, not the scenario |
| `publishedAt` | First publish time |
| `publishedUpdatedAt` | Last time the published snapshot was updated |
| `editedAt` | Last draft edit (`editedAt > publishedUpdatedAt` ⇒ unpublished edits pending) |

- **Non-owners only ever see the published version**, regardless of `viewPublished`.
- All mutations operate on the **draft**; "Save Draft" in the UI writes the draft but does
  not publish. Publishing is a separate, moderated action that updates the snapshot.
- For owner-facing inspection, prefer `viewPublished: false` — it matches the editor and
  the write path.

### No draft lifecycle API

The draft is **implicit and server-managed** — there is no `createDraft` / `ensureDraft` /
`discardDraft` / `revertToPublished` mutation. The client never creates or commits a draft
as a distinct object; it loads state once, edits optimistically, and writes. Consequences:

- "Save Draft" and even "Publish" both go through plain `updateScenario(input)`
  (`UpdateScenarioPublishConfirmation` and `UpdateScenarioModeration` are just aliases for
  it). There's no lighter draft-commit step.
- **Publishing is an AI-moderation pipeline**, not a flag: `aiModerateScenario` (rates text
  and image, returns `final_rating` + explanations, async via `isProcessing`) →
  `confirmAiModeration` → `updateScenario`.
- Because nothing lets you explicitly materialize a draft, there's no way to "pre-save" so
  that a later state read stops forking (the [Practical Gotchas](#practical-gotchas) fork).
  Verified: a full `updateScenario` before the read does not prevent it. The only defense
  is to not read `state(viewPublished:false)` before a card write.

---

## Practical Gotchas

Learned while building the `aid` CLI:

- **Story-card create is an `updateStoryCard` upsert** on a client-chosen `id`. A
  `createStoryCard` mutation exists but the editor's manual "add card" doesn't use it — it
  mints the id itself and upserts. The id formula, from the web bundle, is
  `Math.floor(Math.random() * 1e9).toString()`: a uniform integer in [0, 1e9), no
  uniqueness check (the upsert is scoped by `shortId`). This is why ~10% of real card ids
  are 8 digits rather than 9.
- **Read-before-write card fork**: querying `state(viewPublished: false)` immediately before
  an `updateStoryCard` upsert *on a never-published scenario* makes the new card land only
  in the published view, not the draft — reproducible (5/5), and it does not heal. Cause
  unconfirmed; the read-then-write ordering is what matters. Avoid reading scenario state
  before creating a card (check ownership with a state-free query instead). Full-object
  `updateScenario` and edits to existing cards are unaffected.
- **`updateScenario` is a whole-object replace** — omitted fields are cleared. Always
  round-trip the full state.
- **Soft deletes linger**: `deleteScenario` / `deleteStoryCard` set `deletedAt` but the
  records keep returning from list/option queries; filter client-side.
- **`scenarioStateVersion` is passthrough**, not a concurrency check (doesn't increment).
- **Token refresh is referer-restricted**: the Firebase secure-token endpoint
  (`securetoken.googleapis.com/v1/token`) rejects refreshes without a matching `Referer`
  header (`Requests from referer <empty> are blocked`). Send `Referer:
  https://play.aidungeon.com/` (or the beta host).
- **Profile-search `total` is page size, not a grand total** (see Search → Profile listing).
- **Single-object requests work** even though the web client batches operations as a JSON
  array — `{ query, variables }` is accepted directly.
