# Package collections on atproto

Status: draft, 2026-11-01. Supersedes the sketch in #78; also covers #2890 and the
collection parts of #951 and #2124.

## Thesis

Anyone can make a package collection on any PDS with any client, and if the record uses
our lexicon it shows up on npmx.dev. npmx.dev is one editor and viewer among many, not the
system of record. We design for third-party writers and readers from day one and treat
every record as untrusted input.

## Goals

- Curated, ordered, annotated lists of npm packages, stored in the author's atproto repo.
- Likes and follows on collections.
- Discovery on profiles, on package pages and in a global browse.
- Collections that include other collections.
- No npmx infrastructure on the read path for a single user's collections.

## Non-goals (v1)

- Bookmarks (#80). They're an unbounded, possibly private set, and belong with the separate
  question of where private user settings live.
- Multi-writer collections. atproto records have one owner.
- Flattened or installable views of nested collections.
- Versions. Items reference packages, not package versions.
- Publishing the lexicon. See "Lexicon publishing".

## Data model

### `dev.npmx.collection`

One record per collection, items inline. Record key: TID.

```
name         string   maxGraphemes 64   maxLength 640    required
description  string   maxGraphemes 300  maxLength 3000
items        array    maxLength 1000                      required
  item (object)
    subjectRef  string  format uri        maxLength 300    required
    note        string  maxGraphemes 140  maxLength 560
createdAt    datetime                                     required
```

Only `name`, `items`, `createdAt` and `item.subjectRef` are required. Lexicon evolution
rules never allow relaxing a required field, so everything else stays optional. `item` is
an object rather than a bare string so items can gain fields later.

The NSID is `dev.npmx.collection`, not `dev.npmx.feed.collection`. In `app.bsky`, `feed.*`
is content, `graph.*` is relationships and `actor.*` is profiles; a package collection is
none of these. Non-Bluesky apps also use three segments (`pub.leaflet.document`,
`social.grain.gallery`).

### Why one record, not `collection` + `collection.item`

The official style guide doesn't take a position, but precedent does:

- `app.bsky.graph.list` + `listitem` exists for unbounded membership sets, such as
  moderation lists with thousands of DIDs. We don't need unbounded.
- grain.social split galleries from photos because photos are heavy blobs that can be liked
  on their own. Our items are a URI and a short note.
- plyr.fm playlists are one record with an ordered `items[]` of refs, the same shape as ours.

One record gives ordering for free, one `getRecord` per view, and the simplest target for
third-party writers. The cost is read-modify-write on edits, so third-party writers should
use `putRecord` with `swapRecord` to avoid clobbering.

Typical collections are a few KB, and 1000 bare packages is about 60 KB. Notes are capped
at 560 bytes rather than the usual 1400 for a 140-grapheme field, so even a record with
every note full stays under the 1 MB record cap. The item cap was 100 until collections
also became follow lists, which run larger. A view resolves only its own items, so the
item cap bounds the work per view.

### `item.subjectRef`

`format: uri`, in one of two forms:

- A package: `https://npmx.dev/package/<name>`, the same string `dev.npmx.feed.like` uses
  (`PACKAGE_SUBJECT_REF`). It identifies the package on the network and isn't a dependency
  on npmx.dev. Replacing it with something app-neutral was debated at length and punted.
- Another collection, by at-uri: `at://did:plc:.../dev.npmx.collection/<tid>`.

The lexicon description states both forms and tells consumers to render any other value as
unresolvable rather than reject the collection.

### Likes on collections

No new lexicon. `dev.npmx.feed.like.subjectRef` is `format: uri`, and an at-uri is a URI:

```
dev.npmx.feed.like { subjectRef: "at://did:plc:.../dev.npmx.collection/3abc...", createdAt }
```

Not the `subject` strongRef: it pins a CID, collections are mutable, and every edit would
orphan every like. Bluesky can use strongRef likes because posts don't change. Only the
`subjectRef` description changes, and descriptions aren't constraints.

### Follows

`dev.npmx.graph.follow { subject: at-uri, createdAt }`, modeled on `app.bsky.graph.follow`.
A like is a public endorsement for others; a follow is a subscription for yourself, like
GitHub's star and watch. Only collections are followed. To follow a package, add it to a
collection you follow, so there's no package follow record. Your own collections count as
followed, computed rather than stored. `subject` takes any at-uri, so accounts or other
subjects can be added later without a schema change. Counts come from Constellation, and
the browse index ranks the most followed. Follows are the foundation for a feed and
notifications, which come later.

### Nested collections

The `subjectRef` rule allows nesting at no schema cost. Example: "Vite ecosystem" is Vite
core's own collection plus a community plugins collection someone else curates. This
delegates curation and partly covers the org/collaboration case: the owner adds one
include and the delegate curates from then on.

npmx.dev shows an included collection as a summary (name, author, item count) linking to
its own page, not expanded inline: "Also includes: Vue plugins by @x, 14 packages". The
include is live, so readers need to see whose list a package came from. Not expanding also
means a view's work is bounded by its own items, with no depth limit, cycle detection or
expansion budget. An unresolvable, deleted or hidden include renders as a placeholder.

The lexicon description still tells consumers that do expand inline to stop at 2 levels and
detect cycles.

## Reads

Every read validates the record against the lexicon. Invalid items render as unresolvable;
an invalid record is ignored.

| View                                     | Source                                                                                    | Infra             |
| ---------------------------------------- | ----------------------------------------------------------------------------------------- | ----------------- |
| Profile → Collections                    | `listRecords` on the author's PDS, resolved via `resolveMiniDoc` (same as `getUserLikes`) | none              |
| Collection page                          | `getRecord` on the author's PDS                                                           | none              |
| Package page → "in N collections"        | Constellation backlinks, `dev.npmx.collection:items[].subjectRef`                         | Constellation     |
| Collection → "included in N collections" | same source, target is the collection at-uri                                              | Constellation     |
| Like count, "you liked"                  | Constellation, `dev.npmx.feed.like:subjectRef`, target is the at-uri                      | Constellation     |
| Follower count, "you follow"             | Constellation, `dev.npmx.graph.follow:subject`, target is the at-uri                      | Constellation     |
| Global browse                            | the index below                                                                           | Jetstream + Redis |

Constellation indexes array paths (`app.bsky.feed.post` already shows up under
`facets[].features[app.bsky.richtext.facet#link].uri`) and lags a few seconds, as it does
for likes. Your own profile reads your PDS directly, so your edits show immediately.

Items that don't resolve to a package (typos, unpublished, spam) render greyed out with the
raw name instead of being dropped or failing the collection. The author needs to see them
to fix them.

Everything is public: atproto has no private records and no drafts. The editor says so.

URLs: `/collections/<handle>/<rkey>` maps 1:1 to the at-uri, so a collection written by any
client on any PDS has an npmx.dev URL with no extra step. That's the launch demo.

## Writes

Server-side with the OAuth session, like `server/api/social/like.post.ts`, under new scopes
`repo:dev.npmx.collection` and `repo:dev.npmx.graph.follow`. Existing sessions have to
re-consent; `throwOnMissingOAuthScope` returns a 403 and the client needs to send the user
back through OAuth.

Duplicate reads the record and creates a copy under the current user, with no attribution
field in v1.

The npmx.dev editor is optional. A valid collection doesn't need it.

## Discovery index

Nothing on the network answers "list every `dev.npmx.collection`". A PDS lists one repo,
and Constellation answers "what points at X" only once you know X. Likes don't help, since
counting likes on a collection needs its at-uri first. So global browse needs us to watch
the firehose, and without browse the product has no front door.

- A Vercel Cron Job (`nitro.vercel.config.crons`) calls `/api/cron/collections-index` with
  `CRON_SECRET` every 5 minutes. Nitro's `scheduledTasks` only runs on long-lived servers,
  not the Vercel preset. Each run gets 60 s; replay covers about an hour of history per
  second, so one run can cover Jetstream's whole retention.
- Each run reads the cursor (a Jetstream `seq`) from Upstash, connects to Jetstream v2
  (`jetstream.us-east.bsky.network`) with
  `collections=dev.npmx.collection,dev.npmx.feed.like,dev.npmx.graph.follow` and
  `kinds=commit,identity`, drains until caught up, writes the cursor and exits. Identity
  events (~15/min) act as a clock: replay scans server-side and sends nothing while nothing
  matches, so commits alone can't distinguish "caught up" from "still scanning". Volume is
  tiny; likes arrive a handful per day.
- The Redis index holds `{uri, did, name, itemCount, createdAt}` per collection, plus
  sorted sets for recent, most liked and most followed. Rankings count distinct accounts,
  and only for collections already in the index, so made-up URIs and repeated records can't
  game them. We handle Jetstream `delete` ops ourselves because Constellation is slow to
  drop deleted records.
- Jetstream keeps about 36 hours. An older cursor gets an `OutdatedCursor` info frame and
  resumes at the retention floor, so a longer outage loses events. That's acceptable: the
  PDS is the source of truth and the index only speeds up discovery. Fast follow: a nightly
  reconcile via `listRecords` for each known DID.
- Vercel WebSockets in Functions went GA on 2026-10-20, but that covers inbound connections
  billed as bounded-duration invocations, so it doesn't change this design. For sub-minute
  freshness, a 25-minute drain on a 25-minute cron would work.
- Vercel Queues is in public beta and needs a producer we don't have.

Before the index fills, Bluesky shares help with discovery. Constellation already indexes
`app.bsky.feed.post` links to npmx package URLs, which gives "shared N times on Bluesky" on
package pages with no indexing of our own (built separately on the `bluesky-shares`
branch). The same query works for collection URLs.

## Moderation

npmx is a trusted verifier on Bluesky, but verification is an identity claim. Labeling is
separate: a service account declares `app.bsky.labeler.service`, runs Ozone and emits
`com.atproto.label` labels that clients subscribe to. npmx doesn't run a labeler, and the
vendored `tools.ozone.report` lexicons are unused.

The abuse case is a "top crypto wallets" collection full of malicious packages.

- v1: hide collections whose author DID carries Bluesky moderation labels
  (`com.atproto.label.queryLabels` against `mod.bsky.app`), in both reads and the index.
- Fast follow: an npmx-only denylist of at-uris and DIDs in Redis, filtered at read and in
  the index, plus a "report" button that posts to a Discord webhook or opens a GitHub issue.
  This protects npmx.dev only.
- Later: run Ozone. Labels on at-uris would then protect third-party consumers too.

Collections from an allowlisted set of DIDs get a curated badge, starting with the npmx
account (e18e is a candidate). That's a policy choice; nothing in the record marks
curation.

## Lexicon publishing

Deferred for `dev.npmx.collection` and `dev.npmx.graph.follow`. Until then PDSs can't
resolve their schemas, so third-party writes get no PDS validation and a malformed record
just won't render on npmx.dev. Our reader validates everything regardless.

Publishing means a `com.atproto.lexicon.schema` record per NSID in the npmx account's repo,
plus a `_lexicon.<authority>` DNS TXT record per authority. `dev.npmx.feed.like` and
`dev.npmx.actor.profile` are already published, and `_lexicon.npmx.dev` already exists.
Still needed before launch: the two new schema records, `_lexicon.graph.npmx.dev`, and the
updated `subjectRef` description on the published `like` schema.

## Scope

v1:

- Lexicon JSON and generated types.
- Collection CRUD on npmx.dev; duplicate button.
- Profile → Collections and Followed collections.
- Collection page, with included collections shown as linked summaries.
- Package page → "in N collections".
- Like and follow a collection, with counts.
- Jetstream index; `/collections` sorted by recent, most liked and most followed.
- Bluesky moderation labels on author DID.
- Curated badge for the npmx account.

Fast follow:

- npmx-only denylist and report button.
- Nightly index reconcile.
- "Shared N times on Bluesky" on collection pages.

Later:

- Publish the lexicon; an `@npmx/lexicons` package and a `putRecord` example as the
  third-party SDK.
- Ozone.
- Flattened views, "install all", compare.
- Category pages (#2890) as curated collections from the npmx account.
- `forkedFrom` attribution (an optional field, so non-breaking to add).
- A feed and notifications driven by follows.

## Launch

The demo is one `putRecord` with curl against your own PDS, then the collection's URL on
npmx.dev. Invite people to build their own collection managers; the SDK is the published
lexicon plus a README.

## Decisions log

- Package identity stays `https://npmx.dev/package/<name>`.
- One record with inline items. See "Why one record".
- `dev.npmx.collection`, not `dev.npmx.feed.collection`.
- Likes via `subjectRef` at-uri, never `subject` strongRef.
- Follows target collections only; to follow a package, add it to a collection you follow.
- Up to 1000 items, so collections can serve as follow lists.
- Nested collections allowed, shown as linked summaries, not expanded or flattened.
- Forks without attribution.
- Bookmarks out of scope.
- Lexicon publishing deferred.
- Discovery index is in v1.

## References

- Issues: #78, #80, #951, #1372, #2124, #2890.
- Figma `npmx-dev`: `main-profile-lists` (profile Lists tab, Bluesky user lists; closest
  existing card pattern), `main-profile-likes`. No collections or bookmarks design exists.
- [Lexicon style guide](https://atproto.com/guides/lexinomicon),
  [data validation guide](https://atproto.com/guides/data-validation),
  [pinned posts post-mortem](https://atproto.com/blog/pinned-posts),
  [Constellation](https://constellation.microcosm.blue/),
  [Jetstream replay](https://bsky.network/docs/jetstream-replay),
  [Nitro tasks](https://nitro.build/guide/tasks),
  [Nitro on Vercel](https://nitro.build/deploy/providers/vercel),
  [Vercel WebSocket GA](https://vercel.com/changelog/websocket-support-is-now-generally-available).
- Precedents: `app.bsky.graph.list`/`listitem`, grain.social galleries, plyr.fm playlists.
