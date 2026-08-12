# GM campaign tool — foundation

A web app for running tabletop RPG campaigns. The goal is fast recall and capture during
play, not rules automation. Players use their own paper sheets; this tool is the GM's
memory.

Status: pre-implementation design. Decisions here are settled enough to build on, but the
open questions at the bottom are deliberately unanswered until there's real play experience.

---

## Scope

**In scope**

- Any number of registered users. Anyone can sign up, and each user is the GM of the
  campaigns they create. Standard Laravel auth — registration, verification, password reset.
- Every user sees only their own campaigns. No sharing, no invitations, no way to join
  someone else's campaign.
- Entries: PCs, NPCs, locations, factions, items, lore, sessions, notes, maps.
- Tagging, hierarchy, and free-form linking between entries.
- A persistent party rail with rough health and at-a-glance notes.
- Fast search and fast capture during a session.

**Explicitly out of scope for now**

- Game systems and mechanics. No Daggerheart, no stats engine, no dice.
- Per-round tracking (initiative, hope/fear, stress, conditions).
- Shared campaigns: co-GMs, players joining, invitations, real-time sync.
- Per-user roles inside a campaign. The owner is the only participant.
- GM-composable field definitions / custom sheet schemas.

The last three were designed and then cut. Each is recoverable — see "Deferred, with a way
back" below.

---

## Stack

| Concern | Choice | Note |
| --- | --- | --- |
| Framework | Laravel 13 | PHP 8.3 minimum |
| Frontend | Inertia 2 + Vue 3 + TypeScript | Laravel starter kit ships this preconfigured |
| Styling | Tailwind 4 + shadcn-vue | |
| Database | PostgreSQL | `jsonb` is load-bearing; not optional |
| Search | Laravel Scout, database driver | Postgres full-text is plenty at this scale |
| Media | spatie/laravel-medialibrary | Portraits, maps, handouts |
| Tags | spatie/laravel-tags | Typed tags — see below |
| Markdown | league/commonmark | |
| Types | spatie/laravel-data + spatie/typescript-transformer | |
| Tooling | Pest, Pint, Larastan | |

**Set up `typescript-transformer` on day one.** DTOs generating `types.d.ts` is what keeps
the PHP and TS sides from drifting. Retrofitting it is tedious.

**Why Inertia/Vue over Livewire.** The two hardest screens (link autocomplete, command
palette) want real client-side state. Tradeoff accepted: no Flux, and types live in two
languages.

---

## Data model

### Core principle: everything is an entry

One table for PCs, NPCs, locations, factions, lore, sessions, items, maps. They differ in
presentation, barely in shape — all have a name, a description, tags, and links to other
things.

Giving types their own tables triples the complexity of linking, searching, and tagging,
and forces `union` queries in the command palette. Type-specific fields go in `data jsonb`.

```
users
  # standard Laravel auth; the starter kit's migration is fine as-is

campaigns
  ulid, user_id, name, slug     # unique(user_id, slug)
  premise, active_session_id (nullable FK → entries)
  archived_at, timestamps

entries
  id (bigint PK), ulid (routing), campaign_id
  type (enum)                 # pc|npc|location|faction|item|lore|session|note|map
  parent_id (nullable, self)  # region → town → building
  name, slug                  # unique(campaign_id, slug)
  summary                     # one-liner for cards and hover panels
  body (text)                 # markdown — what the players know
  secrets (text)              # markdown — what they don't
  data (jsonb)                # type-specific extras
  pinned (bool), sort (int)
  archived_at, timestamps

entry_links
  campaign_id, from_id, to_id
  label (nullable)            # "owes", "controlled by", "from"
  secret (bool)               # hidden affiliations
  # unique(from_id, to_id, label)

map_pins
  map_entry_id, target_entry_id
  x, y                        # percentages, never pixels
  label
```

Plus `taggables` from spatie/laravel-tags.

### `data jsonb` contents by type

Keep this small. If a key needs querying in three places, promote it to a real column.

| Type | Keys |
| --- | --- |
| `pc` | `player_name`, `hp_current`, `hp_max` |
| `session` | `number`, `played_at`, `recap` |
| everything else | usually empty |

`player_name` is plain text, not a FK. Correct while there are no logins, and still fine
after — attach a nullable `user_id` alongside it later without migrating existing rows.

### Derived, not stored

- **Stub** = `body` is empty. Powers the prep queue; no flag needed.
- **Backlinks** = `entry_links` where `to_id = this`.
- **Session ordering** = `data->>'number'` descending, until it isn't good enough.

---

## Authorization

Multiple users, but no multi-tenancy: no tenant column on every table, no global tenant
scope, no per-tenant connections. Ownership is a single chain.

```
user → campaigns.user_id → entries.campaign_id → entry_links / map_pins
```

`entries` deliberately has no `user_id`. Denormalising the owner onto every child table is
the beginning of multi-tenancy, and it introduces a second source of truth that can
disagree with the first. Authorise through the campaign instead.

**Resolve the campaign from the route, then never query children globally.**

```php
// routes
Route::scopeBindings()->group(function () {
    Route::get('/campaigns/{campaign}/entries/{entry}', ...);
});

// controller — always through the relation, never Entry::find()
$entry = $campaign->entries()->where('ulid', $ulid)->firstOrFail();
```

With `scopeBindings`, an entry ULID from someone else's campaign 404s before the
controller runs. Combined with a `CampaignPolicy` checking `$campaign->user_id ===
$user->id` and an `authorize` call (or `can:` middleware) on every campaign route, that's
the whole authorization story.

Two things to get right, because they're the places this leaks:

- **ULIDs are unguessable, not authorization.** Every route still authorizes. Obscurity is
  not a check.
- **Search and the command palette are the easiest place to leak.** Scout queries need an
  explicit `where('campaign_id', $campaign->id)` constraint; a global index with no filter
  will happily return other people's NPCs. Test this one deliberately.

A single Pest test asserting that user B gets a 404 on user A's entry, campaign, and search
results is worth writing early — it's the regression that matters most and the one least
likely to be noticed by hand.

Because users are real accounts from the start, adding shared campaigns later is additive:
a `campaign_members` table plus a role, and the policy changes from an ownership check to a
membership lookup. Nothing above has to be undone.

---

## Decisions and rationale

Recording the *why* so these don't get re-litigated later.

**Entry types are a fixed PHP enum.** Adding a case is a one-line change. GM-definable
types were considered and cut — no concrete type was missing from the list.

**No dedicated `faction_id` on entries.** The interesting characters have divided
loyalties: an innkeeper who secretly reports to a faction can't be modelled by a single
FK without one of those facts becoming a lie. Also a slippery slope — `parent_id` is
already one privileged relation; faction would be the second, then owner, then patron.

Affiliation still gets to *feel* first-class: an "Affiliations" UI field reads and writes
`entry_links` rows where the target is type `faction`, supports multiple values, and each
row carries `secret`. Querying members of a faction is one join.

**Sessions are entries, not their own table.** Revisit if `data->>'played_at'` casts start
appearing in several queries; promoting `number` and `played_at` to nullable columns on
`entries` is a cheap migration when that happens.

**No visibility/permission layer yet.** With one table it's one migration and one policy
to add later, so building it now would be premature. But `body` and `secrets` stay
separate fields from the start — framed as revealed vs. hidden *knowledge*, not
permissions. Useful even solo, and already correctly shaped if players ever log in.

**Health is a bar, not a number.** Players own the authoritative sheet; a parallel number
the GM maintains drifts out of sync and then can't be trusted. A draggable bar gives
"is anyone in danger" without owing anyone accuracy. Exact figures appear only in the
hover panel and on the PC page.

**One markdown blob for a PC's stats.** No field definitions, no schema. Rendered when
reading, textarea when editing. This block is also exactly where a typed system sheet
would slot in later, behind the same visual position.

---

## UI

### One shell, no modes

Prep and play were originally two modes. That was wrong: "add a paragraph to an NPC" is
neither, it's just the thing you do. Mode boundaries create friction exactly when you can
least afford it.

Instead: sidebar always present, everything inline-editable always. The party rail and the
capture bar are panels you toggle — left on during a session, collapsed while writing prep.
Same screen, same URLs.

```
┌──────────────────────────────────────────────────────┐
│ campaign name        session 8 live    ⌘K    + new   │
├──────────┬─────────────────────────────┬─────────────┤
│ nav tree │ entry (inline-editable)     │ party rail  │
│ by type, │  badge · breadcrumb         │ (toggle)    │
│ nested   │  name / summary / tags      │ portrait    │
│ where it │  body                       │ + hp bar    │
│ exists   │  hidden block               │ hover for   │
│          │  affiliations               │ detail      │
│          │  linked entries    + new    │             │
│          │  mentioned in               │             │
├──────────┴─────────────────────────────┴─────────────┤
│ + capture: threatened [[Gareth]] #combat          ↵  │
└──────────────────────────────────────────────────────┘
```

### Screens

1. **Campaign home** — premise, counts (entries / stubs / days since last session), a
   prominent "start session N" action, then sessions newest-first with recap lines and
   note counts.
2. **Entry page** — one layout for every type. Badge colour and a couple of `data` fields
   are the only differences.
3. **PC page** — the entry page plus a health row and player name. "Hidden" is labelled
   *hidden from [player]*, because GM-only knowledge about a player's own character is a
   distinct and frequently useful thing.
4. **Map** — image with pins linking to entries.

### Party rail

Collapsed row = portrait, name, health bar. Hover (desktop) or tap-to-pin (touch) expands
in place: larger portrait, player name, exact HP, the sheet blob, affiliation pills with
hidden ones marked, and links to open or edit. Build tap-to-pin from the start rather than
treating touch as an afterthought.

### Creation

Three entry points, one result. All produce a stub: name, type, optional parent. No
required fields, no wizard.

- **`[[` mid-sentence** — the important one. Autocomplete over existing entries; last
  option creates a new one, with inline type chips and the current entry as default
  parent. Shift+Enter creates and returns the cursor to where it was.
- **`+` on a sidebar tree row** — same operation, explicit parent.
- **⌘N** — no context, pick a type.

On save, parse `[[...]]` out of `body` and `secrets`, resolve to entries, create missing
ones as stubs, and sync `entry_links`. This is what actually populates the relationship
graph — nobody fills in an "add relationship" form, but everybody writes names in prose.

### Capture

One always-available input that appends a timestamped line to the active session entry,
accepting `[[links]]` and `#tags`. Over a session it becomes the recap. This plus ⌘K is
most of the tool's value.

### Backlinks are a list, not a graph

Force-directed graphs demo well and are useless at the table. A flat "mentioned in" list
with type icons answers the real question: where does this come up.

---

## Build order

1. Auth from the starter kit, campaigns CRUD, `CampaignPolicy`, scoped route bindings, and
   the cross-user 404 test.
2. `entries` CRUD, markdown body/secrets, tags. One page component for all types.
3. `[[link]]` parsing → `entry_links` → backlinks. Stub creation.
4. Sidebar tree with nesting and per-row create.
5. ⌘K command palette.
6. Campaign home with session list and "start session".
7. Capture bar writing to the active session.
8. Party rail with health bars and hover panel.
9. Maps and pins.

Steps 1–3 are the whole foundation; everything after is affordances on top of it.
