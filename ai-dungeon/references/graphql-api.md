# AI Dungeon GraphQL API Reference

AI Dungeon's web client communicates with a GraphQL API. This documents the API surface
as observed from the Phoenix client. **Not an official public API** — internal endpoints
used by the web client, subject to change.

## Table of Contents
1. [Endpoint and Auth](#endpoint-and-auth)
2. [Key Queries](#key-queries)
3. [Content Model](#content-model)
4. [Search and Discovery](#search-and-discovery)

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
