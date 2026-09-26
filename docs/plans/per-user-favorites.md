# Personal favorites and dislikes: technical plan

Status: agreed with the maintainer, 2026-09-27. Owner: Marcel van der Veldt. This document is
the technical companion of the "Personal favorites and dislikes for every household member"
epic on the project board. It is written so it can be fed to an agent for implementation;
every claim marked "verified" was checked in the referenced code (server `dev` at `bdb867f20`,
models `main` at `6626960`, frontend `main` at `1a1f7ad0`).

## Summary

The `favorite` flag on a library item is one boolean on the shared library row, so one member's
heart is everybody's heart, and until server PR 6212 it was also written into every member's
streaming account. There is no way to say "never play this for me". Favorites become personal
and three-way: each user has their own like, dislike or nothing on every library item, stored
in one new table. Disliked tracks are kept out of playback that Music Assistant generates for
that user. Existing favorites are kept for the members whose accounts hold the item.

Decisions taken with the maintainer (2026-09-27): a per-user table, not a flag on provider
mappings and not a JSON map on the media row; existing favorites map to the owners of the
sources where the item is in their library, household sources map to everyone; the old
`favorite` column is dropped in the same migration; writes that cannot be attributed to a user
go to every user, the rule the play log already follows; one new `music/favorites/set_item`
command, the two old commands stay as wrappers; dislike is stored for every media type and
enforced on tracks first; server PR 6212 merges first and its write-target helper is reused.

This revises one point of the 2026-09-09 music sources decision, which kept favorites global.
Ownership, sharing, the single shared library and shared play counts are untouched.

## Requirements

- A user's like, dislike or unset on any library item, for every media type that can be a
  library item. A user only ever sees their own state; API results and events never leak
  another user's state.
- Provider sync fills in likes reported by a source for that source's owner, never overrides a
  choice the user made, including the choice to remove a like.
- A like or dislike is written back only to the sources the acting user may write to (own and
  visible household sources, the rule of PR 6212).
- Generated playback for a user (radio mode, autoplay, dynamic playlists, mixes) never contains
  a track that user disliked. Explicit playback is untouched.
- Existing installs keep their favorites: a like for the owners of the sources where the item
  is in their library, a like for everyone where it comes from a household source.
- The migration is idempotent, survives half-written data and never raises (a failed library
  migration resets the database).

## Today (verified)

- `favorite BOOLEAN NOT NULL DEFAULT 0` on all eight media tables, one index each
  (`music_assistant/controllers/music/database.py:299-432`, `585-599`). Nothing else references
  the column: the timestamp and FTS triggers do not.
- `MediaControllerBase.set_favorite` is `@final`, keyed on the library id, emits
  `MEDIA_ITEM_UPDATED` (`controllers/music/media/base.py:1067-1077`).
- Listings filter with `{table}.favorite = :favorite` (`base.py:2004-2008`), counts with
  `favorite = 1` (`base.py:415-435`, `artists.py:143`, `albums.py:309`, `genres.py:296`).
  Hydration in `_parse_db_row` (`base.py:2133`) and `_parse_summary_row` (`base.py:2363`).
- Sync can only turn the flag on (seven sites in `models/music_provider.py`, e.g.
  `1342-1344`) and clears it once when an item left every provider library (`1130-1134`).
  Inserts write `item.favorite` (eight `_add_library_item` sites), updates leave it alone,
  merge ORs it (`base.py:2652-2661`), the genre update clobbers it (`genres.py:1374`).
- Users live in `auth.db`; `webserver.setup()` runs after the music controller, so a library
  migration cannot list users (`mass.py:288-328`). A post-webserver one-off precedent exists
  right after `webserver.setup()` in `mass.start()`.
- The only per-user library data: `playlog.userid` and `playlists.access` (`database.py:274-290`,
  `playlists.py:1873-1890`). `mark_played` without a user writes a row for every user
  (`controllers/music/controller.py:1688-1695`).
- Providers writing favorites back: Apple Music (rating `1`/`-1`, `apple_music/library.py:225-236`),
  Plex (like/unlike rating, `plex/__init__.py:1117-1139`), OpenSubsonic (star/unstar,
  `opensubsonic/sonic_provider.py:395-416`). Plex already reads a tri-state from its rating
  threshold (`plex/__init__.py:1508-1510`).
- Generated playback converges on `RadioPlaylistProvider.get_dynamic_tracks`
  (`providers/radio_playlist/__init__.py:116-208`) and the queue loader fills
  (`controllers/player_queues/queue_loader.py:511-548`, `1053-1116`, `1065-1133`), which already
  know the queue's user. None of them reads `favorite`.

## Architecture

### 1. Models (music-assistant/models)

- `MediaItem.favorite: bool | None = None` (`media_items/media_item.py:231`). `True` like,
  `False` dislike, `None` unset. Old clients testing truthiness keep working.
- `EventType.FAVORITE_UPDATED = "favorite_updated"`. Payload: `{"user_id", "media_type",
  "item_id", "uri", "favorite"}`, `object_id` = the item uri. Clients apply it only when
  `user_id` is their own.
- Trap for the server sweep: any code writing `favorite = False` now writes a dislike. Every
  provider parser that writes `bool(starred)` or a bare `False` must write `True if x else None`.

### 2. Server storage (schema 61)

New table, created in `__create_database_tables` and in the migration:

```sql
CREATE TABLE IF NOT EXISTS favorites(
  [user_id] TEXT NOT NULL,
  [media_type] TEXT NOT NULL,
  [item_id] INTEGER NOT NULL,
  [favorite] BOOLEAN,
  [timestamp] INTEGER NOT NULL DEFAULT 0,
  UNIQUE(user_id, media_type, item_id));
CREATE INDEX IF NOT EXISTS favorites_item_idx ON favorites(media_type, item_id);
```

`favorite` 1 is a like, 0 a dislike, NULL a row the user explicitly reset. No row means the user
never expressed anything. The NULL row exists so that provider sync, which only ever
`INSERT OR IGNORE`s, can neither flip a dislike nor re-heart something the user removed.

A small `FavoritesStore` in `controllers/music/favorites.py` (shape of `recency.py`) owns the
table and is reachable as `mass.music.favorites`: `get(user_id, media_type, item_id)`,
`set(user_ids, media_type, item_id, favorite)`, `record_from_provider(instance_id, media_type,
item_id, favorite)`, `disliked_track_keys(user_id)`, `move_item`, `remove_item`,
`release_user(user_id)`, `expand_pending()`.

Migration step `prev_version <= 60`, in this order:

1. Create the table and index.
2. For every media table and every configured music source with an owner (raw provider configs
   via `helpers/provider_access.py`, readable at migration time): `INSERT OR IGNORE` a like for
   the owner on every favorited item that has an `in_library = 1` mapping on that instance.
3. For every favorited item that has an `in_library = 1` mapping on a household source (no
   access record or owner `None`), or no `in_library = 1` mapping at all: `INSERT OR IGNORE` a
   like under the placeholder user id `__pending__`.
4. `DROP INDEX IF EXISTS {table}_favorite_idx`, then `ALTER TABLE {table} DROP COLUMN favorite`,
   swallowing "no such column". Remove the column and index from `__create_database_tables`.

Post-webserver one-off in `mass.start()`, next to the local_audio tombstone: `expand_pending()`
copies every `__pending__` row to every user from `auth.list_users()` (`INSERT OR IGNORE`, one
statement per user) and deletes the placeholder rows. Idempotent on data presence, no flag.

Sweeps: `remove_item_from_library` deletes the item's rows; `_merge_library_items` moves the
source item's rows to the target with `INSERT OR IGNORE` then deletes them, ordered so a crash
between the two statements loses nothing; `auth.delete_user` calls `release_user` next to
`release_user_playlists` (`webserver/auth.py:1268`). The genre update stops writing `favorite`.

### 3. Server reads

- Hydration: `_build_final_query` (`base.py`, `@final`) adds
  `LEFT JOIN favorites AS fav ON fav.media_type = :media_type AND fav.item_id = {table}.item_id
  AND fav.user_id = :favorite_user_id` and selects `fav.favorite AS favorite`; the bound user id
  is the current user, or a value that never matches when there is none (internal callers see
  `None`). The per-controller base and summary queries stop selecting the dropped column;
  `_summary_base_columns` and the two per-type summary parsers (`tracks.py:1677`,
  `audiobooks.py:678`) read the joined column.
- Filter: `_apply_filters` replaces the column test with
  `EXISTS(SELECT 1 FROM favorites WHERE user_id = :favorite_user_id AND media_type = ... AND
  item_id = {table}.item_id AND favorite = :favorite)`. `favorite=True` lists likes,
  `favorite=False` dislikes, `None` no filter; the parameter keeps its type. The random subquery
  path reuses the same clause. `library_count(favorite_only)` uses the EXISTS form.
- `LibraryItemSyncDetails` loses `favorite`.
- `set_favorite` (`base.py`) becomes `set_favorite(item_id, favorite, user_ids)`: upserts a row
  per user (`NULL` row on unset), invalidates the search results cache like `_store_access`,
  emits one `FAVORITE_UPDATED` per user. It no longer emits `MEDIA_ITEM_UPDATED`.
- The "recently favorited tracks" recommendation row orders on `favorites.timestamp`
  (new sort key) instead of `timestamp_modified`.

### 4. Server writes and attribution

- New `music/favorites/set_item(item, favorite: bool | None)` (scope `LIBRARY_WRITE`).
  `music/favorites/add_item` sets `True`, `music/favorites/remove_item` sets `None`; both stay.
- Attribution, one helper used everywhere: the session user; else every user from
  `auth.list_users()`. `players/add_currently_playing_to_favorites` sets the queue's playback
  user as current user before calling (the queue loader already does this).
- Write-through: `MusicProvider.set_favorite(prov_item_id, media_type, favorite: bool | None)`.
  Default mapping in the three overriding providers: like stars, unset and dislike unstar
  (OpenSubsonic); Apple Music writes rating `1`, deletes the rating on unset, `-1` on dislike;
  Plex writes the like rating, clears on unset, the unlike rating on dislike. Targets come from
  `_write_target_mappings` (PR 6212). Add awaits inline, remove and dislike stay fire-and-forget.
- Sync: the seven `if not favorite and prov_item.favorite` sites become
  `record_from_provider(self.instance_id, media_type, db_id, prov_item.favorite)` when the
  provider item carries a state: `INSERT OR IGNORE` for the instance owner, or for every user
  when the instance is a household source. The deletion sweep clears the item's likes (rows
  with 1) once no provider holds it in a library; dislikes survive.
- Parser sweep to `True if x else None`: Apple Music (`library.py:363`, `parsers.py:213,304,346`),
  Jellyfin (`parsers.py:143,179,276,301`), Emby (`parsers.py:137,185,256,300`), OpenSubsonic
  (`parsers.py:149,288,363`), filesystem (`__init__.py:2182,2198`). Deezer and iBroadcast write
  `True` already; Plex's `get_favorite_from_rating` already yields the tri-state.
- API schema version bump (new command, new event, changed field type).

### 5. Server: generated playback

`disliked_track_keys(user_id)` runs one query per queue fill and returns the user's disliked
track library ids plus the `(provider_domain, provider_item_id)` pairs of their mappings.
`filter_disliked(tracks, keys)` drops a candidate by library id or by any of its mappings.
Applied, keyed on the queue's `userid`, in `_fill_dynamic_tracks`, `_fill_autoplay_music_tracks`
(after the recency gate) and `_get_similar_tracks`. An anonymous queue is not filtered. It is a
hard filter, unlike the advisory `TrackFilter` (`helpers/track_filter.py`), which stays as is.

The builtin "all favorited tracks" and "infinite mix from favorites" playlists and the smart
playlist `favorites_only` rule resolve against the current user through the existing context;
when a queue fills from them the loader has restored the playback user
(`queue_loader.py:520-527`). Artist and album level gating is a follow-up.

### 6. Frontend

- `favorite: boolean | null` in `interfaces.ts:1003-1008`.
- The heart stays a like toggle (`FavoriteButton.vue`, `DetailHeroFavorite.vue`,
  `InfoHeader.vue`, `MediaRowList.vue`). Dislike and unset become entries in the context menu
  (`ItemContextMenu.vue:756-843`) and the player-bar favorite dropdown (`FavoriteMenuBtn.vue`).
  A disliked item shows a thumbs-down badge in rows and the detail hero. shadcn and lucide or
  tabler icons only, no new Vuetify.
- `api.toggleFavorite` keeps flipping like and unset; new `api.setFavorite(item, state)` calls
  `music/favorites/set_item`. The `getLibrary*` signatures are unchanged (an off toggle already
  sends `undefined`, `LibraryTracks.vue:110`).
- Events: subscribe to `FAVORITE_UPDATED`, apply only when `user_id` is the signed-in user.
  Every `MEDIA_ITEM_UPDATED` whole-item replacement (`ItemsListing.vue:1982-1998`,
  `TrackDetails.vue:223-249` and the other detail views, `useShortcuts.ts:446-463`) keeps the
  local `favorite`, because the broadcast item carries the acting user's state. One shared
  helper for that merge.
- Screenshots in the PR, per the frontend review rules.

### 7. Docs

Favorites page on music-assistant.io: favorites are yours, the dislike, what generated
playback does with it, what happens to existing favorites on update.

## Rollout

1. Server PR 6212 (write-through scoped to the acting user's sources) merges first.
2. Models: nullable favorite and the event. Release, then bump the server pin.
3. Server storage (schema 61): sections 2, 3 and 4, with tests for the migration (owned,
   household and unmapped favorites), the pending expansion, per-user listings and counts,
   sync attribution, write-through and events.
4. Server playback gate: section 5, with tests on the three fill paths.
5. Frontend: section 6.
6. Docs: section 7.

## Follow-ups

- Apple Music and Plex dislike polish, YouTube Music dislike write-back.
- Artist and album level dislike gating in generated playback.
- A "disliked" filter chip in the library views.
- Home Assistant favorite action (backlog #8), local file rating tags (backlog #12).
