# Tags: your own lists over the whole library: technical plan

Status: agreed with the maintainer, 2026-11-06. Owner: Marcel van der Veldt. This document is
the technical companion of the "Tags: your own lists over the whole library" epic on the
project board. It is written so it can be fed to an agent for implementation; every claim
marked "verified" was checked in the referenced code (server `dev` at `1351aae65`, models
`main` at `7484f2a`, frontend `main` at `d0f901ce`, python client `main` at `3ff6e01`).

## Summary

A tag is a label a user makes and puts on anything in the library: an artist, an album, a
track, a playlist, a radio station, an audiobook or a podcast. "Listen later", "Dinner",
"Kids", "Vinyl I own". A tag lists what carries it, per media type, with the usual search,
sort and paging; every library listing can filter on tags; a tag plays as a queue; a tag can
be shared with the household; a smart playlist can be built from a tag; an assistant can list,
fill and play tags. Each user gets a "Listen later" tag out of the box that empties itself as
its items get listened to.

Tags are the user's own organisation. Nothing about them syncs to or from a music source, and
a library sync never touches them. That is what sets them apart from genres (which describe the
music and come from the sources and MusicBrainz, with aliases) and from favorites (a per-user
state that syncs both ways with the streaming accounts).

Demand (support discussions, read 2026-11-06): the maintainers already treat "tags" and
"listen later" as one request (#5023 closed as a duplicate of #1559 "custom tags"); three
separate people asked for a Roon-style Listen Later list in eight months (#5075, #6747 by a
maintainer, #6914), and the gap they all name is that playlists cannot hold albums; scene and
mood labels on albums, tracks, playlists and radio that can be filtered in the UI and played
from Home Assistant come up repeatedly (#2761, #1559, #2419, #4466, #6745); people abuse custom
genres as labels today (#6080). Star ratings (#2461), ordered curated collections (#6603) and
playlist folders (#3039) are different shapes and stay out.

Decisions taken with the maintainer (2026-11-06): a tag is a library-only media type modelled
on Genre, not a side table; a tag has an owner and sharing exactly like a Music Assistant
playlist and is private by default; tagging an item that is not in the library adds it to the
library first, as favoriting does, with no write-through to the source; the UI word is "Tags";
favorites and dislikes stay what they are; the first version ships manual tags with colour and
icon, a per-tag "remove after played" switch, a seeded "Listen later" tag per user named in
the system language, a smart playlist rule, MCP tools, the python client and docs; smart tags
(rule-based membership) are a feature issue under the epic, built right after, so every
membership read goes through one place now.

## Requirements

- Any user with `library.write` can create tags and tag or untag any library artist, album,
  track, playlist, radio station, audiobook or podcast, one at a time or as a selection.
- A user sees their own tags, the tags shared with them and the household's tags, never
  another user's private tag: not in listings, search, chips, counts or events.
- A tag's contents list per media type with search, sort and paging; "date tagged" is a sort
  field and the default order, newest first.
- Every library listing filters on one or more tags, combined with the filters it has
  (favorite, genre, provider, album type, search).
- A tag is playable: `play_media` on its uri plays its members in tag order, each member as it
  plays on its own. A tag is addressable by name (voice, assistants).
- A tag can drop an item once the user listened to it (the "remove after played" switch).
- Every user gets a "Listen later" tag, named in the system language, that they may rename or
  delete; it never comes back on its own.
- Tags never sync to or from music sources. An item leaving the library leaves its tags, a
  merge moves membership, a deleted user's tags fall to the household.
- The migration is idempotent, survives half-written data and never raises.
- Smart playlists can use tags as a rule; assistants can list tags, tag items and play a tag.

## Today (verified)

- Genre is a library-only media type with no provider mappings (`controllers/music/controller.py:1871`):
  the `genres` table and `genre_media_item_mapping(genre_id, media_id, media_type)`
  (`controllers/music/database.py:481-514`), FTS and timestamp triggers through
  `MEDIA_ITEM_DB_TABLES` (`constants.py:290-299`), a genre filter on every library listing as an
  EXISTS clause in `_apply_filters` (`controllers/music/media/base.py:2642`, genre at 2662-2672,
  threaded through `_apply_random_subquery` at 2580 and the collapse-collections branch at
  2119), an overview of one `RecommendationFolder` per media type
  (`controllers/music/media/genres.py:650-698`), per-type counts (700-735), `music/genres/tracks`
  and `albums` (423, 462). The frontend genre page lists the other media types through the
  library listings with the genre filter (`views/GenreDetails.vue:345-389`). Playback resolves a
  genre to tracks (`controllers/player_queues/media_resolver.py:171, 268, 1163`) and credits the
  container once (`_mark_container_played`, 1429).
- Favorites are a per-user side table owned by `FavoritesStore` (`controllers/music/favorites.py:54`).
  `music/favorites/set_item` (`controller.py:1626`) resolves a uri or item, refuses audio sources
  and sound effects and adds the item to the library when needed. Rows leave with the item
  (`base.py:483`), move on merge (3661) and are released when a user is deleted
  (`controllers/webserver/auth.py:1458-1460`). `_favorite_filter_clause` (`base.py:2396`) and
  `_favorite_sort_sql` (2414) serve the filter and `SortField.FAVORITE_TIMESTAMP`.
- MA playlists carry `PlaylistAccess(owner, sharing, shared_users, collaborative)`
  (`music_assistant_models/access.py:26`) as json on the row. Visibility is the
  `_listing_filter_clause` override (`controllers/music/media/playlists.py:2269`, the only override
  of `base.py:2542`); `set_access` (836) validates owner and share list; `release_user_playlists`
  (900) hands a deleted user's playlists to the household; `visible_to_caller`, `_check_visible`
  and `_check_may_manage` (948, 2306, 2321) gate reads and edits; the update command is
  re-registered to refuse non-managers (958); `_new_playlist_access` (2359) makes a household
  playlist for the HA system user; `access_allows` lives in `helpers/provider_access.py:38`.
- Playlists hold playable items only (`PLAYLIST_SUPPORTED_MEDIATYPES`,
  `music_assistant_models/media_items/media_item.py:37-45`): no albums, artists or podcasts. That
  is the gap every listen-later request names.
- Translations: `mass.translations.get_translation(key, locale)`
  (`controllers/translations/__init__.py:85`) resolves a key from the per-locale bundles with the
  English source as fallback; `media.*` keys live in `strings.json:501`. The system language is
  `mass.metadata.locale` (`controllers/metadata/controller.py:246`), set once from the first client
  or by the user.
- Users: `auth.create_user` (`controllers/webserver/auth.py:409`) with the first-user hook at 468.
  Post-webserver one-offs run in `mass.start()` right after `migrate_provider_access`
  (`mass.py:368-376`, the favorites `settle_pending` precedent).
- Play reports: `controllers/player_queues/playback_tracker.py:772-797` forwards a fully played
  item to `mark_item_played` and credits an album once the first of its tracks played from it
  completes (`_claim_album_credit`, 862). `QueueItemOrigin.container`
  (`music_assistant_models/queue_item.py:25`) names the album, playlist, folder, podcast or
  audiobook a queue item was played from; `played_from_album` is at
  `controllers/player_queues/helpers.py:205`.
- Smart playlists: `SmartPlaylistRules` (`providers/smart_playlist/helpers.py:99`) with
  `genre_ids`/`genre_names` and `favorites_only`, evaluated with the user context
  (`providers/smart_playlist/__init__.py:384, 880`).
- MCP: the builtin server has `get_favorites`, `add_favorite`, `remove_favorite`,
  `get_playlists`, `add_to_playlist` and `play`, which resolves a uri or a name
  (`providers/mcp_server/tools.py:70-93, 226-290`); the fastmcp server maps command prefixes to
  capabilities (`providers/fastmcp_server/command_policy.py:184`, the `music/` catch-all) and marks
  item arguments (`target_filters.py:95-115`).
- Frontend: library entries in `MENU_ITEM_REGISTRY`
  (`components/navigation/utils/getMenuItems.ts:114-176`); pinned shortcuts keyed by uri with
  `SUPPORTED_TYPES` (`composables/useShortcuts.ts:32-41`); genre chips in
  `components/details/DetailHeroGenres.vue`; `LinkGenreDialog.vue` opened from the context menu
  (`layouts/default/ItemContextMenu.vue:1465`); multi-select in `components/ItemsListing.vue:840`;
  listing filters in `composables/useItemsListing.ts:30-47` and `ItemsListing.vue:1790-1900`;
  `MediaType.GENRE` is handled in 21 files (the sweep a new media type needs);
  `helpers/media_type_icon.ts:26`.
- Python client: `get_library_genres`, `get_genre_tracks`, `get_genre_albums`
  (`music_assistant_client/music.py:943-1037`), schema gates in `constants.py`.
- `DB_SCHEMA_VERSION = 71` (`controllers/music/constants.py:15`), latest step `<= 70`
  (`controllers/music/migrations.py:1407`). `API_SCHEMA_VERSION = 103` (`constants.py:73`).

## Architecture

### 1. Models (music-assistant/models)

- `MediaType.TAG = "tag"`, in `MediaType.ALL` next to `GENRE`: search, counts and play history
  include tags; no provider claims to support the type, so a library sync skips it.
- `ItemAccess(ProviderAccess)` carries `collaborative`; `PlaylistAccess(ItemAccess)` keeps its
  name with no fields of its own. The wire format does not change.
- `Tag(MediaItem)`: `media_type = TAG`, `access: ItemAccess | None` (None = the household's),
  `color: str | None` (CSS hex), `icon: str | None` (lucide icon name),
  `remove_after_played: bool = False`, `item_count: int | None = None` (filled by the server on
  gets and listings). No translation key: a tag's name is the user's. Added to `MediaItemType`,
  `media_from_dict` and `SearchResults.tags`.
- `SortField.TAG_TIMESTAMP = "tag_timestamp"` (when the item was tagged).
  `ListingType.LIBRARY_TAGS` (the tags in the library) and `ListingType.TAG_ITEMS` (the items
  of a tag, any media type).
- `EventType.TAG_ITEMS_UPDATED = "tag_items_updated"`, `object_id` = the tag uri, payload
  `TagItemsUpdate(tag_id, media_type, item_ids, added)`. The tag row itself goes through
  `MEDIA_ITEM_ADDED/UPDATED/DELETED` like any library item.
- Release, then bump the server pin.

### 2. Server storage (schema 72)

Two tables, created in `__create_database_tables` and by the migration step `prev_version <= 71`
(`CREATE TABLE IF NOT EXISTS`, nothing to move, idempotent):

```sql
CREATE TABLE IF NOT EXISTS tags(
  [item_id] INTEGER PRIMARY KEY AUTOINCREMENT,
  [name] TEXT NOT NULL,
  [sort_name] TEXT NOT NULL,
  [search_name] TEXT NOT NULL,
  [search_sort_name] TEXT NOT NULL,
  [metadata] json NOT NULL,
  [access] json,
  [color] TEXT,
  [icon] TEXT,
  [remove_after_played] BOOLEAN NOT NULL DEFAULT 0,
  [play_count] INTEGER NOT NULL DEFAULT 0,
  [last_played] INTEGER NOT NULL DEFAULT 0,
  [timestamp_added] INTEGER DEFAULT (cast(strftime('%s','now') as int)),
  [timestamp_modified] INTEGER NOT NULL DEFAULT 0);
CREATE TABLE IF NOT EXISTS tag_items(
  [tag_id] INTEGER NOT NULL,
  [media_type] TEXT NOT NULL,
  [item_id] INTEGER NOT NULL,
  [timestamp] INTEGER NOT NULL DEFAULT 0,
  UNIQUE(tag_id, media_type, item_id));
CREATE INDEX IF NOT EXISTS tag_items_item_idx ON tag_items(media_type, item_id);
```

`tags` joins `MEDIA_ITEM_DB_TABLES` (FTS index, timestamp triggers). No provider mappings, like
genres. New `DB_TABLE_TAGS` and `DB_TABLE_TAG_ITEMS` constants.

Sweeps: `remove_item_from_library` deletes the item's `tag_items` rows next to the favorites
call; `_merge_library_items` moves them (`INSERT OR IGNORE` then `DELETE`, in that order);
removing a tag deletes its rows; `_cleanup_database` drops rows whose tag or item is gone;
`auth.delete_user` calls `tags.release_user(user_id)` next to the playlists and favorites calls.
The playlist release logic (`playlists.py:900-937`: owned items become the household's, the
user leaves every share list, a selected-share item nobody could see any more becomes the
household's) moves into one function on the access record that both controllers call. Small
refactor, same behaviour.

### 3. TagsController and API (`controllers/music/media/tags.py`)

`TagsController(MediaControllerBase[Tag])`, `db_table = tags`, registered in `MusicController`
(`self.tags`, `get_controller(MediaType.TAG)`, the api command tuple). `match_providers` is a
no-op; `_add_library_item` and `_update_library_item` write the row; `refresh_item` returns
early as it does for genres (`controller.py:1871`). The generic `music/tags/count`,
`library_items` (with FTS search), `get` and `remove` come from the base.

Who may do what, the MA playlist rules verbatim: see = household tag, or owner, or on the
share list; edit items = owner, or on the share list of a collaborative tag; manage (rename,
colour, icon, switch, share, delete) = owner or `library.manage`. Internal callers without a
user see everything. Implemented with the playlist pieces on the tag row: the
`_listing_filter_clause` override, `visible_to_caller`, `_check_visible`, `_check_may_manage`,
`check_removal_allowed`, and `music/tags/update` re-registered with `library.write` plus the
manage check.

Commands (`library.write` unless noted):

- `music/tags/create(name, color=None, icon=None, remove_after_played=False) -> Tag`. Owner =
  the session user, private; a household tag for the HA system user or no user. The name is
  trimmed, non-empty and unique among the tags the caller owns (case-insensitive).
- `music/tags/set_access(item_id, sharing, owner=None, shared_users=None, collaborative=False)
  -> Tag`: `set_access` of playlists (836-898) on the tag row, user validation included.
- `music/tags/add_items(tag_id, items: list[str | MediaItemType | ItemMapping]) -> None` and
  `music/tags/remove_items(tag_id, items) -> None`. Each entry is a uri or an item. The media
  type must be one of the seven taggable types (artist, album, track, playlist, radio,
  audiobook, podcast); anything else raises `InvalidDataError`. An item without a library row is
  added with `add_item_to_library` (no provider write-through, same as favorites). Rows go in
  with `INSERT OR IGNORE` and the current time, or out with `DELETE`; the tag's
  `timestamp_modified` is bumped; the search results cache is invalidated; one
  `TAG_ITEMS_UPDATED` per media type and one `MEDIA_ITEM_UPDATED` for the tag are emitted.
  Idempotent.
- `music/tags/items(tag_id, media_type, favorite=None, search=None, limit=50, offset=0,
  sort_field=None, sort_direction=None) -> list[MediaItemType]` (`library.read`): the type
  controller's `get_library_items_by_query` with the membership clause and the
  `ListingType.TAG_ITEMS` sort options: `TAG_TIMESTAMP` descending first (the default), then
  the fields every library type shares (name, sort name, date added, date modified, play count,
  last played, favorite timestamp), narrowed by `MEDIA_TYPE_SORT_FIELDS`. Per-type extras
  (album year, artist) stay on the library views, which filter by tag.
- `music/tags/overview(tag_id) -> list[RecommendationFolder]` (`library.read`): one folder per
  media type that has members, newest first, the shape of `GenreController.get_overview`.
- `music/tags/media_counts(tag_id) -> dict[MediaType, int]` (`library.read`).
- `music/tags/tags_for_media_item(media_type, item_id) -> list[Tag]` (`library.read`): the
  visible tags holding the library item, one query, for the chips.
- `music/tags/library_items` and `get` fill `item_count` with a correlated count.

Listing filter and sort: `tag_ids: list[int] | None` on `library_items` of every media
controller and on `get_library_items_by_query`, applied in `_apply_filters` as
`EXISTS(SELECT 1 FROM tag_items ti WHERE ti.media_type = :tag_media_type AND ti.item_id =
{table}.item_id AND ti.tag_id IN :tag_ids)` and threaded through the random subquery and the
collapse-collections branch like `genre`; `library_count` takes it too. A tag id the caller may
not see raises `MediaNotFoundError`. `SortField.TAG_TIMESTAMP` orders on a correlated subquery
like `_favorite_sort_sql`, offered on `TAG_ITEMS` only.

Smart-tags preparation: `TagsController.membership_clause(tag_ids, media_type, params)` is the
one builder of that EXISTS clause, used by `_apply_filters`, `items`, `overview`,
`media_counts`, `tags_for_media_item` and the smart playlist rule. The follow-up replaces it per
tag with a rules-derived clause and refuses `add_items`, `remove_items` and
`remove_after_played` on a smart tag. Nothing else is reserved: a nullable `rules` column is one
guarded `ALTER TABLE` later.

API schema version bump: new media type, commands, listing parameter, sort field, event.

### 4. Playback

- `media_resolver`: `MediaType.TAG` resolves to its members in tag order (date tagged ascending,
  first in first out; a `sort_field`/`sort_direction` given to `play_media` wins), each member
  through its own existing resolution: a track to itself, an album to its tracks, a playlist,
  artist or genre as it plays on its own, a radio station to itself, an audiobook to its resume
  point, a podcast to its next episode. Members keep their own origin (an album played from a
  tag looks like an album played directly), the tag is credited once with
  `_mark_container_played`. `get_tracks_for_playback` (171) follows for radio mode and the quiz.
  Unavailable members are skipped.
- `play_media` accepts `library://tag/<id>`; `parse_uri` needs nothing beyond the enum.
  `music/item_by_name(name, media_type=TAG)` finds a tag by name through the library search,
  preferring the caller's own tag when names collide.
- Remove after played, one hook in `playback_tracker` next to `_claim_album_credit`: on a
  fully played report, when the played item (a track or an audiobook) or the queue item's
  `origin.container` (an album, playlist, artist or podcast) is a member of a tag with the
  switch on that the queue's user can see, and no later queue item carries that same origin,
  `tags.on_member_played(media_type, library_id, user_id)` removes the membership. A radio
  station never leaves on its own (it has no end). The tag lookup costs one query per fully
  played report and only while at least one tag has the switch on (a cached flag).

### 5. Seeded "Listen later"

- `strings.json`: `media.tag.listen_later.name = "Listen later"`, translated through the usual
  bundles.
- `TagsController.seed_user_tags(user)` creates the tag for a user who does not own one with
  that name: name from `translations.get_translation("media.tag.listen_later.name",
  locale=mass.metadata.locale)` with the English source as fallback, `icon = "clock"`,
  `remove_after_played = True`, private. Called from `auth.create_user` for every user except
  the HA system user, and once for all existing users from a post-webserver one-off in
  `mass.start()` next to `settle_pending()`, gated by a `tags_seeded` row in the library
  `settings` table so a user who deleted the tag never gets it back. Never raises (a warning).

### 6. Smart playlist rule

`SmartPlaylistRules` gains `tag_ids`, `tag_names`, `excluded_tag_ids`, `excluded_tag_names`,
the genre pattern. A track matches when it is tagged itself or its album is tagged, evaluated
in the SQL path with `membership_clause` in the user context the playlist already uses, so a
tag the evaluating user cannot see contributes nothing. Artist-level expansion is a later
option. Frontend: tag pickers in the smart playlist editor. dmoo500 owns that area and hears
about it when the server part lands.

### 7. MCP

Builtin MCP server (`providers/mcp_server/tools.py`): `get_tags` (visible tags with counts),
`get_tag_items(tag, media_type=None)`, `tag_item(tag, uri)`, `untag_item(tag, uri)`,
`create_tag(name)`; `play` gets `tag` in `SearchMediaType` and `_resolve_media` resolves a tag by
name through `music/tags/library_items(search=...)`. fastmcp server: `music/tags/` is covered
by the `music/` family; `TARGET_RULES` mark the `items` argument of `add_items` and
`remove_items` as music references.

### 8. Frontend

- Types and wrappers: `MediaType.TAG`, `Tag`, `ItemAccess`, the listing types,
  `SortField.TAG_TIMESTAMP` and `EventType.TAG_ITEMS_UPDATED` in `plugins/api/interfaces.ts`;
  wrappers in `plugins/api/index.ts`; the `MediaType.GENRE` sweep (icons, quiz sources,
  progressive search, context menu, shortcuts, favorites helper, radio helper).
- Tags library (`/tags`, `views/LibraryTags.vue`): `ItemsListing` with `ListingType.LIBRARY_TAGS`;
  a row shows icon and colour, name, "n items" and a shared or household badge; a Create button;
  an empty state that points at the Listen later tag and the "Add to tag" menu entry.
- Tag page (`/tags/:itemId`, `views/TagDetails.vue`): hero with icon, colour, counts, Play,
  favorite and a menu (Edit, Share, Delete for managers); one row per media type from
  `music/tags/overview`, each "show all" opening a per-type listing driven by `music/tags/items`
  with the `TAG_ITEMS` sort options; inside a tag page the context menu offers "Remove from
  tag".
- Tagging: "Add to tag…" in `ItemContextMenu.vue` for the seven taggable types, library and
  provider items alike, also for a multi-selection; `components/tags/TagPickerDialog.vue` lists
  the tags the user may edit as checkboxes with their state for the item(s) and a "New tag"
  inline; `components/tags/CreateEditTagDialog.vue` (name, colour swatches, a small lucide icon
  set, the remove-after-played switch); sharing through the playlist share dialog generalised
  over `ItemAccess`.
- Chips: `components/details/DetailHeroTags.vue` next to the genre chips on all seven heroes; a
  chip opens the tag page; refreshed on `TAG_ITEMS_UPDATED`.
- Filters: `tagIds` in `LoadDataParams`, a Tags entry in the `ItemsListing` filter menu
  (`show-tag-filter`) on all seven library views, options from `music/tags/library_items`,
  remembered like the genre filter.
- Sidebar: "Tags" in the library group of `MENU_ITEM_REGISTRY`; `MediaType.TAG` in
  `useShortcuts` `SUPPORTED_TYPES` so a tag can be pinned, drawn with the tag's own icon and
  colour; `media_type_icon.ts`.
- Events: `MEDIA_ITEM_*` for tags update the list and the page; `TAG_ITEMS_UPDATED` updates
  chips, the tag page and counts.
- Screenshots in the PR, per the frontend review rules.

### 9. Python client

`music.py`: `get_library_tags`, `get_tag`, `create_tag`, `update_tag`, `remove_tag`,
`set_tag_access`, `add_tag_items`, `remove_tag_items`, `get_tag_items`, `get_tag_overview`,
`get_tag_media_counts`, `get_tags_for_media_item`; `tag_ids` on every `get_library_*` call,
gated by a `TAGS_SCHEMA_VERSION` constant.

### 10. Docs

`tags.md` on music-assistant.io: what a tag is and is not (file tags, genres, favorites),
Listen later and the remove-after-played switch, sharing, tagging from the menu and in bulk, the
tag page, filtering any listing by tag, playing a tag, pinning, the smart playlist rule,
assistants. A line in `genres.md` sends people who use custom genres as labels to tags.

## Testing

Server, with the database fixtures: the migration on a fresh database and from schema 71;
seeding (new user, existing users once, HA system user skipped, system language, no return
after deletion); create, update, remove with the permission matrix (private, selected,
everyone, collaborative, library.manage); `add_items` adding a provider item to the library and
refusing an episode, a genre, a tag; the tag filter on every media type including `count`, the
random path and collapsed collections; `TAG_TIMESTAMP` sorting; overview, counts and chips;
the sweeps (item removed, items merged, tag removed, user deleted); playback resolution of a
mixed tag; remove-after-played for a track, an album whose last track completes, an audiobook
and a radio station that never leaves; the smart playlist rule in the user context; the MCP
tools. `pytest -n auto --dist loadfile` and `pre-commit run --all-files` green.

Frontend: vitest on the access helper; screenshots of the library, the page, the picker and a
filtered listing in the PR.

## Sub-issues

1. Models: tag media type, item access, sort field, listing types, event.
2. Server: tag storage, controller, API, sharing, sweeps and the seeded Listen later (schema 72).
3. Server: tag filter and sort on every library listing, tag items, overview and counts.
4. Server: play a tag, remove after played.
5. Server: smart playlist rule "tagged with".
6. Server: MCP tools for tags.
7. Frontend: tags everywhere (library, page, dialogs, chips, filters, sidebar, events).
8. Frontend: tag rule in the smart playlist editor.
9. Client: tags in the python client.
10. Docs: tags page.

Feature issue under the epic, built right after: Smart tags (rule-based membership).

## Later

- Smart tags: a tag with `rules` (a saved filter per media type over the library listing
  filters, and the smart playlist rule fields where they fit), membership computed by
  `membership_clause`, no manual add or remove, no remove-after-played, chips computed on the
  detail page only. The feature issue under the epic holds the sketch.
- Home Assistant core integration: `tag` as a media type for `play_media` and the media browser
  (other repository).
- Artist-level expansion in the smart playlist rule.
- Favorites and Disliked as read-only rows in the Tags view, if people ask for one place.
- Export and import of tags next to playlist export.
- Writing tags into the metadata of local files.

## Changes during implementation

None yet.
