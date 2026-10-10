# Server-side sorting, search and paging for every listing: technical plan

Status: proposal 2026-10-10. Owner: Marcel van der Veldt. This document is the technical
companion of the "Sort, search and page every listing on the server" epic (#272) on the project
board. It is written so it can be fed to an agent for implementation; every claim marked
"verified" was checked in the referenced code (server at `ab52d428c`, `music-assistant-models`
1.1.218 at `0394e19`, frontend `main` at `eb3def25`, python client at `21b5f55`, mobile app at
`a501ff18`, Home Assistant core `dev` of 2026-10-10).

## Summary

Only the library listings (`music/<type>s/library_items`) sort, search, filter and page on the
server. Every other listing, such as the tracks of an album or playlist, an artist's albums,
podcast episodes, versions, similar items, genre contents and browse folders, returns the
whole list and leaves sorting, searching and filtering to the client. The web frontend does
that in `ItemsListing.vue`, and every other client (mobile app, Home Assistant, the python
client) has to reinvent it. Playlist tracks and podcast episodes are streamed as async
generators chunked at 500 items per websocket message, which every client has to reassemble.
The order a user sees is not the order that plays: `play_media` carries its own legacy
`sort_by` string and re-sorts with a second, frontend-shaped implementation.

The fix gives every listing command the same contract: `search`, typed `sort_field` and
`sort_direction`, `limit` and `offset`, plus the filters that listing supports. Library-backed
listings keep pushing it into SQL; container listings assembled from providers run one shared
in-memory pipeline on a cached full listing. A single `music/sort_options(listing)` command
tells clients which sorts a listing offers and which one is the default. Playback resolves a
container through the same pipeline, so the queue mirrors the listing. The contributor PRs by
dmoo500 (models#356 and #417, merged; server#5498 and frontend#2928, open) are the first
steps and are finished as part of this epic.

## Decisions

Taken with the maintainer on 2026-10-10:

- Server PR 5498 is finished by us: we push a cleanup to dmoo500's branch (typed sort passed
  through the server internals, the legacy `order_by` strings parsed once at the API boundary,
  no second `field:direction` string grammar, `SortOptionInfo` reused instead of a server copy,
  the thirty internal callers moved to the typed parameters, YEAR for collapsed album
  collections), then merge. The frontend PR 2928 is adjusted the same way. Both get a short
  comment from Marcel explaining the pushes.
- Sort options are discovered through one command, `music/sort_options(listing)`, keyed on a
  `ListingType` enum in the models package; the first option returned is the listing's
  default. The per-media-type `music/<type>s/get_sort_options` from PR 5498 is replaced before
  it ships.
- `playlist_tracks` and `podcast_episodes` become coroutines returning one page, `limit`
  defaulting to 500 like `library_items`. Outdated clients silently get the first 500 items.
  The websocket chunking of async-generator results stays in the server for now, unused by
  these two commands, and is removed later.
- The full assembled listing of a container is cached on the controller level (short TTL,
  bypassed by `force_refresh`, invalidated on playlist edits), on top of the provider-level
  `use_cache` that about half of the providers have.
- `music/browse` is part of the epic: search, sort (the provider's order stays available as
  `ORIGINAL`) and paging, as its own sub-issue after the shared pipeline exists.
- Playback mirrors the listing: plain Play on an album or playlist page sends the sort the
  listing shows, not only "play from here". Play from elsewhere uses the listing's default.
- Models change first (new sort fields and the listing enum, released and pinned) so the
  server work never carries a server-only sort vocabulary.

## Requirements

- Every listing command accepts `search`, `sort_field`, `sort_direction`, `limit`, `offset`
  and the filters that make sense for it, with one meaning everywhere.
- A client can ask which sorts a listing offers, their default direction and the default sort.
- Paging, re-sorting and searching a container never re-query the provider while the cached
  listing is fresh; `force_refresh` still does.
- The queue plays in the order the listing shows when the client asks for it.
- Outdated clients keep working: legacy `order_by` and `sort_by` strings keep their meaning,
  `playlist_tracks` and `podcast_episodes` answer with their first page.
- No provider has to change. `get_playlist_tracks(page)`, `get_album_tracks` and the other
  provider methods keep their contracts.
- Nothing becomes a way to download or keep provider audio; listings carry media items, not
  stream URLs, exactly as today.

## How it works today

Verified in the server at `ab52d428c` unless noted.

**Library listings.** `MediaControllerBase.library_items`
(`controllers/music/media/base.py:585-668`) takes `favorite`, `search`, `limit=500`,
`offset=0`, `order_by="sort_name"`, `provider`, `genre`, `played_only`, `summary`,
`collapse_collections`, `reachable_via`; the subclasses add `album_types`, `explicit`,
`artist_type`, `album_artists_only`, `hide_empty`, `media_type`, `content_type`. `order_by` is a
legacy string looked up in `SORT_KEYS` (`base.py:155-185`, 25 keys such as
`timestamp_added_desc`, `album_artist_name`, `random_play_count`) plus the per-user
`favorite_timestamp` keys (`base.py:186`, `_favorite_sort_key` at `:2038-2048`), applied in
`_build_final_query` (`:2355-2385`), with a sampled subquery for the random keys
(`_apply_random_subquery`, `:2208-2240`) and a reduced key set for collapsed collections
(`_adapt_query_for_collections`, keys at `:2827-2849`, applied at `:2942-2946`). Thirty server-internal callers pass these
strings (recommendations, builtin, autoplay, msx bridge, metadata, smart playlist, media
resolver). The genre contents commands `music/genres/tracks` and `albums`
(`media/genres.py:400-456`) already take `limit`, `offset` and `order_by`.

**Container listings.** All return everything, unsorted beyond their natural order:

| command | method | returns | order |
|---|---|---|---|
| `music/albums/album_tracks` | `albums.tracks` (`media/albums.py:399-497`) | list | disc and track number, sorted at `:494-497` |
| `music/playlists/playlist_tracks` | `playlists.tracks` (`media/playlists.py:189-237`) | async generator | provider pages, in order |
| `music/podcasts/podcast_episodes` | `podcasts.episodes` (`media/podcasts.py:158-175`) | async generator | provider order |
| `music/artists/artist_tracks`, `artist_albums` | `artists.tracks`, `albums` (`media/artists.py:240-282`) | list | SQL default (library) or provider order |
| `music/artists/artist_appears_on`, `discography`, `top_tracks`, `top_albums`, `artist_audiobooks`, `similar_artists` | `media/artists.py:284-470` | list | newest first, provider ranking, or SQL default |
| `music/tracks/track_versions`, `track_albums`, `similar_tracks` | `media/tracks.py:418-500` | list | search order |
| `music/albums/album_versions`, `podcast_versions`, `audiobook_versions`, `radio_versions` | `versions` on each controller | list | search order |
| `music/browse` | `MusicController.browse` (`controllers/music/controller.py:833`) | list | provider folder order |

Playlist tracks are fetched page by page from the provider (`_get_provider_playlist_tracks`,
`playlists.py:1531-1557`, under `guard_single_request` and `cache.handle_refresh`); only the
provider's own `use_cache` decorator caches the pages, and 16 of the 36 providers that
implement `get_playlist_tracks` carry one directly above the method. The websocket handler
collects async-generator results and sends a `partial=True` message every 500 items
(`controllers/webserver/websocket_client.py:313-325`); the REST handler materialises the
whole list (`webserver/controller.py:811-815`). The python client reassembles partial results
(`music_assistant_client/client.py:338-356`, `get_playlist_tracks` at `music.py:367-385`), the
frontend does the same (`src/plugins/api/index.ts:3061-3074`), and the mobile app documents it
in `composeApp/.../api/Paging.kt`.

**Frontend.** `ItemsListing.vue` has two paths (`src/components/ItemsListing.vue:482-485`):
`loadPagedData` for the eight library views, which send `sortBy` as `order_by`, and `loadItems`
for every detail listing, which loads everything once and runs `getFilteredItems`
(`:2240-2390`): substring search over name, album name and first artist name; sorts for
`name`, `sort_name`, `album`, `album_sort_name`, `artist`, `track_number`, `position`,
`timestamp_added`, `year`, `duration`, `provider` and their `_desc` variants; favourites,
hide-fully-played and album-type filters; then an offset/limit slice. Twenty-one views use the
component; the detail pages pass their own `sort-keys` (`PlaylistDetails.vue:76-86`,
`AlbumDetails.vue:28-34`, `ArtistListing.vue:87-105`, `PodcastDetails.vue:30-36`,
`BrowseView.vue:11`, `RadioDetails.vue:20`). Select-all pages through a paged listing with
`loadAllItems` (`:1138-1146`, `:2393-2425`).

**Playback.** `play_media` (`controllers/player_queues/controller.py:516-564`) takes
`sort_by: str | None`, handed through `queue_loader.py:760,969` to
`media_resolver.get_album_tracks` (`media_resolver.py:161-202`) and `get_playlist_tracks`
(`:294-354`), which re-sort in memory with `sort_tracks` (`player_queues/helpers.py:166-210`,
its own key map: `position_desc`, `name`, `artist`, `album`, `duration`, `timestamp_added`,
`track_number`). The frontend sends `sort_by` only for "play from here"
(`src/helpers/media_item_actions.ts:63-85`, `src/layouts/default/ItemContextMenu.vue:1426-1460`).

**Other clients.** Home Assistant's media browser reads library listings with `limit=500,
order_by=SORT_NAME` and the whole playlist through `get_playlist_tracks`
(`homeassistant/components/music_assistant/media_browser.py:224,253`). The mobile app pages
`library_items` with legacy `order_by` strings and `SERVER_PAGE_SIZE = 500`, and keeps its own
`LibraryFilters` of the server filters it found to work.

**dmoo500's PRs.** models#356 added `SortField` (12 members, `enums.py:1035-1049`) and
`SortDirection`; models#417 added `SortOptionInfo(field, supports_direction,
default_direction, label_key)` (`api.py:18-24`); both are in 1.1.218, which the server pins.
Server#5498 (head `ddb7db47b`, CI green, no open threads, 77 tests pass locally) adds
keyword-only `sort_field`/`sort_direction` to every `library_items`, a per-media-type allowed
field table (`controllers/music/sorting.py`), `_get_sort_sql` overrides for `ARTIST_NAME` on
tracks and albums, `music/<type>s/get_sort_options`, random-paging fixes and
`tests/controllers/music/test_sorting.py`. It still encodes the typed parameters into a
`"field:direction"` string (`_resolve_sort_parameters`) that `_build_final_query` parses back
(`_parse_order_by`), so the string parsing you asked to drop in August survived as a second
grammar; `SortFieldDefinition` duplicates `SortOptionInfo`; `BASE_SORT_FIELD_SQL` maps
`RANDOM_PLAY_COUNT` to the wrong clause (special-cased later); collapsed collections strip the
table qualifier with `str.replace` and fall back to NAME for YEAR. Frontend#2928 adds
`LibrarySortControls.vue`, the `useLibrarySorting` composable and `getLibrarySortOptions`, but
sends the `"name:desc"` preference string as `order_by` and never uses the typed parameters.

## 1. Models: sort fields and listing types

In `music-assistant-models`:

- `SortField` gains `TRACK_NUMBER` (disc then track number), `ALBUM_NAME` (a track's album),
  `PROVIDER` (the item's provider display name, for versions listings), `FAVORITE_TIMESTAMP`
  (when the calling user liked it; replaces the server-only `favorite_timestamp` keys) and
  `ORIGINAL` (the order the source lists them: a provider's playlist, folder or ranking order,
  a discography newest first). `ORIGINAL`, `RANDOM` and `RANDOM_PLAY_COUNT` carry no direction.
- A `ListingType` `StrEnum` names every listing a client can ask sort options for:
  `LIBRARY_ARTISTS`, `LIBRARY_ALBUMS`, `LIBRARY_TRACKS`, `LIBRARY_PLAYLISTS`, `LIBRARY_RADIOS`,
  `LIBRARY_AUDIOBOOKS`, `LIBRARY_PODCASTS`, `LIBRARY_GENRES`, `ALBUM_TRACKS`, `ARTIST_ALBUMS`,
  `ARTIST_TRACKS`, `ARTIST_APPEARS_ON`, `ARTIST_DISCOGRAPHY`, `ARTIST_TOP_TRACKS`,
  `ARTIST_TOP_ALBUMS`, `ARTIST_AUDIOBOOKS`, `SIMILAR_ARTISTS`, `SIMILAR_TRACKS`, `TRACK_ALBUMS`,
  `VERSIONS` (track, album, podcast, audiobook and radio versions share one option set),
  `PLAYLIST_TRACKS`, `PODCAST_EPISODES`, `GENRE_TRACKS`, `GENRE_ALBUMS`, `BROWSE`.
- `SortOptionInfo` stays as merged. The python client's `play_media` and listing methods take
  the new parameters in sub-issue 9.

Released as a patch version; the server pin is bumped in sub-issue 2.

## 2. Typed sort through the server (finishing server#5498)

Pushed to dmoo500's branch `feat/sort-field-definitions`, keeping his commits:

- `library_items` keeps `sort_field`, `sort_direction` and the deprecated `order_by`. The
  boundary parses `order_by` once with `LEGACY_SORT_KEYS` (`controllers/music/constants.py`,
  extended with `favorite_timestamp(_desc)` and kept as the only legacy map) into the typed
  pair; from there `get_library_items_by_query`, `iter_library_items`,
  `_localized_search_fallback`, `_apply_random_subquery`, `_build_final_query` and
  `_adapt_query_for_collections` take `sort_field: SortField | None` and
  `sort_direction: SortDirection | None`. `_parse_order_by`, `_resolve_sort_parameters` and the
  `"field:direction"` grammar are removed. `FAVORITE_TIMESTAMP` is generated by the existing
  per-user subquery.
- `_get_sort_sql(field, direction)` stays the per-type SQL hook, fed from `BASE_SORT_FIELD_SQL`
  with the `RANDOM_PLAY_COUNT` entry corrected; collapsed collections get their own unqualified
  clause builder and `YEAR` (`MIN(year)` over the collection) instead of the NAME fallback.
  Answer to dmoo500's question: the generic API allows collapsed album queries, so YEAR is
  handled there too.
- `sorting.py` keeps the field definitions as `SortOptionInfo` instances (no
  `SortFieldDefinition`); its per-media-type table becomes the `LIBRARY_*` rows of the
  listing table in section 4, and `music/<type>s/get_sort_options` becomes
  `music/sort_options(listing)` (section 4) in the same PR, so no client ever sees the
  per-type command.
- The thirty internal callers pass `sort_field=SortField.RANDOM_PLAY_COUNT` and friends.
  `webserver/auth.py` orders a different table and is untouched.
- Tests: `test_sorting.py` loses the parser tests and gains legacy-map and typed-plumbing
  tests; `test_library_listing_queries.py` keeps its legacy-string calls as the compatibility
  check; `API_SCHEMA_VERSION` is bumped once.

## 3. One listing contract and the shared pipeline

Every listing command takes the same core parameters, appended after its existing ones so
positional callers keep working:

| parameter | type | meaning |
|---|---|---|
| `search` | `str \| None` | case- and accent-insensitive substring match on name, album name and artist names (the same normalisation as `search_name`) |
| `sort_field` | `SortField \| None` | `None` is the listing's default |
| `sort_direction` | `SortDirection \| None` | `None` is the field's default direction for that listing |
| `limit` | `int \| None = 500` | `None` returns everything (internal callers, explicit client choice) |
| `offset` | `int = 0` | |

Listing-specific filters keep their existing names where they exist and follow the
`library_items` naming otherwise: `favorite: bool \| None` (needs the calling user's state,
stamped with `with_user_favorites` from `controllers/music/favorites.py:301` generalised to all
playable item types), `album_types: list[AlbumType] \| None` (artist albums), `fully_played:
bool \| None` (podcast episodes, tri-state like `favorite`), `in_library_only` (album tracks,
exists), `provider_filter` (artist listings, exists).

Two execution paths, chosen per listing and never mixed:

- **SQL** for the listings that can be arbitrarily large and live entirely in the database:
  `library_items`, `music/genres/tracks` and `albums`. They use `get_library_items_by_query`
  with the typed sort from section 2.
- **In-memory pipeline** for container listings, which are bounded by the container and are
  assembled in Python anyway (library rows merged with provider rows for album tracks and
  artist audiobooks, provider pages for playlists, search results for versions): a new
  `controllers/music/listing.py`
  with `apply_listing(items, listing, *, search, sort_field, sort_direction, limit, offset,
  filters)` that filters, then sorts with one key function per `SortField` (names through the
  `search_name` normalisation, `TRACK_NUMBER` as `(disc_number or 1, track_number or 0)`,
  `TIMESTAMP_ADDED` from `date_added` with missing dates oldest, `ORIGINAL` keeps the input
  order, Python's stable sort keeps ties in listing order), then slices. `resolve_sort(listing,
  sort_field, sort_direction)` validates against the listing table and raises
  `InvalidDataError` for a field the listing does not offer, mirroring the SQL path.

`sort_tracks` in `player_queues/helpers.py` is deleted in sub-issue 6; its callers use
`apply_listing`.

## 4. Sort options discovery

`music/sort_options(listing: ListingType) -> list[SortOptionInfo]`, registered once on the
music controller, backed by `LISTING_SORT_OPTIONS: dict[ListingType, tuple[SortOptionInfo,
...]]` in `listing.py`. The first entry is the listing's default; `default_direction` is per
listing, so podcast episodes default to `POSITION` descending while playlist tracks default to
`POSITION` ascending. The proposed table (library rows as in PR 5498, with SORT_NAME first as the default):

| listing | options, default first |
|---|---|
| `LIBRARY_ARTISTS`, `LIBRARY_PLAYLISTS`, `LIBRARY_RADIOS`, `LIBRARY_PODCASTS`, `LIBRARY_GENRES` | SORT_NAME, NAME, TIMESTAMP_ADDED, TIMESTAMP_MODIFIED, LAST_PLAYED, PLAY_COUNT, FAVORITE_TIMESTAMP, RANDOM, RANDOM_PLAY_COUNT |
| `LIBRARY_ALBUMS` | the above plus YEAR, ARTIST_NAME |
| `LIBRARY_TRACKS` | the above plus DURATION, ARTIST_NAME |
| `LIBRARY_AUDIOBOOKS` | the above plus DURATION |
| `ALBUM_TRACKS` | TRACK_NUMBER, NAME, ARTIST_NAME, DURATION |
| `PLAYLIST_TRACKS` | POSITION, TIMESTAMP_ADDED, NAME, ARTIST_NAME, ALBUM_NAME, DURATION |
| `PODCAST_EPISODES` | POSITION (desc), NAME, DURATION |
| `ARTIST_TRACKS` | SORT_NAME, NAME, ALBUM_NAME, DURATION |
| `ARTIST_ALBUMS`, `ARTIST_AUDIOBOOKS` | SORT_NAME, NAME, YEAR |
| `ARTIST_APPEARS_ON`, `ARTIST_DISCOGRAPHY`, `TRACK_ALBUMS` | ORIGINAL (newest first), NAME, SORT_NAME, YEAR |
| `ARTIST_TOP_TRACKS`, `SIMILAR_TRACKS` | ORIGINAL (ranking), NAME, DURATION |
| `ARTIST_TOP_ALBUMS`, `SIMILAR_ARTISTS` | ORIGINAL (ranking), NAME |
| `VERSIONS` | ORIGINAL, NAME, PROVIDER |
| `GENRE_TRACKS`, `GENRE_ALBUMS` | as `LIBRARY_TRACKS` and `LIBRARY_ALBUMS` |
| `BROWSE` | ORIGINAL (folder order), NAME |

The frontend's `label_key` lookup (`sort.<label_key>`) and the translations added by PR 2928
keep working; new fields get their keys in sub-issue 7.

## 5. Listing cache

Container listings that come from a provider (playlist tracks, podcast episodes, provider
album tracks, provider artist tracks and albums, versions, similar, top items) are assembled
once and cached as a whole in the in-memory tier of `mass.cache`:

- key: the listing type, the provider instance, the provider item id and, where the user's
  provider filter narrows the assembly (album tracks of a library album), the allowed
  instances; category `CACHE_CATEGORY_LISTINGS` next to `CACHE_CATEGORY_SEARCH_RESULTS`
  (`controllers/music/constants.py:19`); expiration 15 minutes; not persisted.
- `force_refresh=True` bypasses it through the existing `cache.handle_refresh` (which sets
  `BYPASS_CACHE`, `controllers/cache/controller.py:389-395`), so the provider-level
  `use_cache` is bypassed in the same request as today.
- `add_playlist_tracks` and `remove_playlist_tracks` delete the playlist's entry where they
  signal `MEDIA_ITEM_UPDATED` (`playlists.py:1835,2004`); `refresh_item` deletes the item's
  entries. Dynamic playlists and radio stations are never cached (`is_dynamic`, fresh sample
  per call, as today).
- Favourite state is stamped per calling user after the cache read, so a cached list never
  carries another user's likes (the pattern the builtin provider already uses,
  `providers/builtin/__init__.py:562-563`).

Paging, re-sorting and searching a 5000-track playlist then costs one provider walk every
15 minutes instead of one per page.

## 6. Playlist tracks and podcast episodes as pages

`playlists.tracks` and `podcasts.episodes` become coroutines with the section 3 contract
(`limit` default 500) and the listing cache. The generator bodies move to internal
`iter_tracks` and `iter_episodes` methods, not registered as commands, used by the consumers
that stream everything: `media_resolver.get_playlist_tracks` fast path (`:316-335`, skip until
`start_item` without materialising), `metadata/enrichment.py:436`,
`providers/playlist_metadata/__init__.py:253,573`, `msx_bridge/http_server.py:1238,1303,2194`,
`music_quiz/quiz_types/base.py:337`, `ai_radio/runtime.py:439`, the playlist controller's own
edit, export and migrate flows (`playlists.py:839,1234,1606,1642`),
`models/music_provider.py:1852`, `media_resolver.py:413,493`.
`allow_dynamic_tracks` and `strict_provider_instance` keep their meaning. The websocket
chunking code stays for any remaining generator command and goes in the Later cleanup.

## 7. The remaining listings

Album tracks, artist tracks, albums, appears-on, discography, top tracks and albums,
audiobooks, similar artists and tracks, track albums, the five versions listings, the genre
contents commands and browse adopt the contract: library-backed ones through SQL, the rest
through `apply_listing` on the cached assembly. `albums.tracks` keeps its disc/track sort as
the `TRACK_NUMBER` default and keeps returning the merged library-plus-provider list with the
`in_library_only` switch. `browse` keeps `path` and `player_id`, applies search and sort to
the folder's items and pages them; folders sort before items within `NAME`. Legacy `order_by`
on the genre commands maps through the same legacy map as section 2.

## 8. Playback follows the listing

`play_media` gains `sort_field` and `sort_direction`; `sort_by` stays, deprecated, and is
mapped once at the boundary through the legacy map extended with the frontend's keys
(`artist`, `album`, `track_number`, `position_desc`). `_resolve_media_items`,
`get_album_tracks` and `get_playlist_tracks` in the media resolver ask the controller for the
sorted full listing (`limit=None`) instead of re-sorting; `start_item` and
`keep_preceding_items` apply after that as today; the no-sort fast path keeps using
`iter_tracks`. `sort_tracks` and its key map are removed. An invalid field for the container
raises `InvalidDataError` before anything is enqueued.

## 9. Frontend

Two steps. First frontend#2928 is finished on dmoo500's branch: the API wrappers send
`sort_field` and `sort_direction` (never a `"field:direction"` `order_by`), `getLibrarySortOptions`
becomes `getSortOptions(listing)`, the `useLibrarySorting` composable keeps its
`"field:direction"` string only as the saved preference and splits it for the request,
`LibrarySortControls.vue` and the chips stay as designed, the library views pass their
`ListingType`. Then `ItemsListing.vue` keeps only the paged path: `loadItems`, `allItems`,
`getFilteredItems` and the `sortKeys` prop go; `LoadDataParams` carries `sortField`,
`sortDirection`, `search`, `favoritesOnly`, `fullyPlayed`, `albumType`, `provider`,
`libraryOnly`, `limit`, `offset`; every detail view (`AlbumDetails`, `PlaylistDetails`,
`ArtistDetails`, `ArtistListing`, `TrackListing`, `PodcastDetails`, `PodcastEpisodeDetails`,
`GenreDetails`, `CollectionDetails`, `RadioDetails`, `BrowseView`) hands the params to its API
call; select-all pages with the server page size; "play" and "play from here" on album and
playlist pages pass the listing's `sortField`/`sortDirection`; the stale
`no-server-side-sorting` attributes go.

## 10. Python client, mobile app, Home Assistant

The python client gets `sort_field`/`sort_direction` on every listing method (legacy
`order_by` kept, deprecated), `get_sort_options(listing)`, the contract on
`get_playlist_tracks`, `get_podcast_episodes`, `get_album_tracks` and the artist listings, and
the typed sort on `play_media`; the models pin follows the release. Home Assistant needs no
change to keep working and sees the first 500 playlist tracks until it pages. The mobile app
can drop its partial-result handling and its own sort vocabulary when it adopts the contract;
both are Later items for their own repositories.

## 11. What stays

The provider contracts, the library database schema, the `library_items` filters and their
semantics, `MIN_SCHEMA_VERSION`, the user access rules on playlists, the readrate pacing and
every other usage-policy guard are untouched. Legacy `order_by` and `sort_by` keep working for
one release cycle after the clients are updated.

## Worked examples

- **Album page sorted by duration.** The frontend asks `music/sort_options(listing=
  "album_tracks")`, shows TRACK_NUMBER, NAME, ARTIST_NAME, DURATION, the user picks duration
  descending; the page calls `music/albums/album_tracks(item_id, provider, sort_field=
  "duration", sort_direction="desc", limit=50, offset=0)`; Play sends `play_media(album,
  sort_field="duration", sort_direction="desc")` and the queue starts with the longest track.
- **A 3000-track Spotify playlist on the mobile app.** The first `playlist_tracks(limit=500)`
  walks the provider pages (through Spotify's own 3-hour `use_cache`), stores the assembled
  list for 15 minutes and returns 500; the next pages, a search for "live" and a re-sort by
  artist are served from the cache without a provider call.
- **Home Assistant before it is updated.** `get_playlist_tracks(item_id, provider)` receives
  one message with the first 500 tracks instead of six partial messages.
- **An old frontend build.** `library_items(order_by="timestamp_added_desc")` maps through
  the legacy table to `TIMESTAMP_ADDED` descending; `play_media(sort_by="artist")` maps to
  `ARTIST_NAME` ascending.

## Testing

- Extend `tests/controllers/music/test_sorting.py` (legacy map, typed plumbing, every
  `LIBRARY_*` option reaches SQL without a warning, collapsed collections with YEAR) and
  `test_library_listing_queries.py` (the legacy-string calls stay as the compatibility check).
- New `tests/controllers/music/test_listing.py`: table-driven `apply_listing` for every
  `SortField`, both directions, search normalisation, stability of ties, `limit=None`,
  offsets past the end, filters, and `resolve_sort` rejecting a field the listing lacks.
- New `tests/controllers/music/test_listing_cache.py`: second call serves from the cache,
  `force_refresh` and a playlist edit do not, dynamic playlists are never cached, favourite
  state follows the calling user.
- Extend `tests/controllers/music/test_album_tracks_*.py` and the artist listing tests with
  the contract; new tests for the paged `playlist_tracks` and `podcast_episodes`, and for
  `music/sort_options` and the `ListingType` coverage (every listing type has a table row,
  every row's fields exist for that listing's item type).
- Extend `tests/controllers/player_queues/` media resolver tests: typed sort, legacy `sort_by`
  mapping, `start_item` after sorting, invalid field rejected.
- Frontend: `tests/components/ItemsListing.test.ts` and `tests/composables/useLibrarySorting.test.ts`
  from PR 2928, extended for the single paged path and the play-with-sort calls.
- Manual: album and playlist pages on web and mobile layout, a Spotify playlist above 1000
  tracks, podcast episodes with hide-played, browse a filesystem folder, Home Assistant media
  browser against the new server.
- `pre-commit run --all-files` and `pytest -n auto --dist loadfile` green per PR.

## Sub-issues

One PR each, in this order; 4 and 5 can run in parallel after 3, 7 after 2.

| # | issue | sections | size |
|---|---|---|---|
| 1 | #273 Models: new sort fields and the listing types | 1 | tiny |
| 2 | #274 Server: typed sort for library listings (finish server#5498) | 2, 4 | medium |
| 3 | #275 Server: one listing contract, the shared pipeline and the listing cache (album tracks first) | 3, 4, 5 | large |
| 4 | #276 Server: artist, versions, similar, genre and browse listings on the contract | 7 | medium |
| 5 | #277 Server: playlist tracks and podcast episodes as pages | 6 | medium |
| 6 | #278 Server: playback follows the listing sort | 8 | small |
| 7 | #279 Frontend: server-driven sorting for the library views (finish frontend#2928) | 9 | medium |
| 8 | #280 Frontend: every listing sorts, searches and pages on the server | 9 | large |
| 9 | #281 Client: listing contract and sort options in the python client | 10 | small |
| 10 | #282 Docs: the listing contract, sort options and the deprecations | 3, 4, 8 | small |

## Later

- Remove legacy `order_by` and `sort_by`, the websocket partial chunking and the clients'
  partial-result code once Home Assistant, the mobile app and the python client have adopted
  the contract.
- Per-user default sort and view preferences stored on the server (discussion 6004).
- Mobile app: sort options, search and paging for container listings.
- Library tracks sorted by album name (needs an album join in the SQL path) if asked for.
- Sort options for recommendation and home rows are out of scope; they are curated lists.

## Changes during implementation

- 2026-10-10, #274 (server#5498): an unknown legacy `order_by` key raises `InvalidDataError`
  instead of being silently dropped; every key the web frontend, the mobile app and Home
  Assistant send to `library_items` is in the legacy map.
- 2026-10-10, #274: a direction sent with RANDOM or RANDOM_PLAY_COUNT is ignored rather than
  rejected; a direction without a field applies to the listing default.
- 2026-10-10, #274: the library listings offer SORT_NAME first, so the UI default matches the
  server's implicit default and the mobile app; a web user without a saved sort preference sees
  "Sort name" instead of "Name".
- 2026-10-10, #274: until the models release with FAVORITE_TIMESTAMP is pinned (part B of the
  issue), the `favorite_timestamp` legacy keys ride on a separate carrier through the internals
  and the recommendations' recent-favourites row keeps its legacy key.
- 2026-10-10, noticed in #274, not fixed there: the msx_bridge "recently played" handlers sort
  `last_played` ascending (least recently played first); converted faithfully, the fix is a
  separate PR.
