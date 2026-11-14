# Items by external id, resolved on demand: technical plan

Status: proposal 2026-11-14, design decisions taken (see Decisions). Owner: Marcel van der
Veldt. This document is the technical companion of the "Items by external id: top tracks and
discography that are only looked up when opened or played" epic (#452) on the project board.
Every claim marked "verified" was checked in the referenced code (server `dev` at `4e2c937d6`,
`music-assistant-models` 1.1.238, frontend `main` and mobile-app `main` of 2026-11-14). Claims
marked "assumption" were not checked.

## Summary

Music Assistant can list an album it has not found on any music service (the discography), but
only for MusicBrainz release groups and only through a pseudo provider named `musicbrainz`.
Tracks have no such form, so the top tracks fallback resolves 25 Last.fm titles by searching
every one on the user's services, which is being cut back to library-only matching.

This plan adds one generic form for an item known only by an external id, a resolver that turns
it into a real item when the user opens, adds or plays it, a ListenBrainz-ranked top tracks list
built on it, lazy resolution in the queue, and moves the discography onto the same form right
away. No database or model change, no new setting, no background job.

## Decisions

Taken with the maintainer on 2026-11-14:

- Form: `external://<type>/<id_type>:<value>` with `provider="external"`. Generic for every id
  type, no collision with the real `musicbrainz` provider, no models change.
- The discography switches at once, in the same release as the resolver and the frontend. It
  does not wait for a mobile app release; `musicbrainz://album/<id>` stays a permanent alias.
- Top tracks list source: ListenBrainz without a token now (one request per artist, cached 7
  days, a 401 means "no list"). In parallel we ask MetaBrainz whether the project can get a
  token, and look into serving the popularity data from our own mirror.
- List length: 50.
- Playing the whole list: the tracks are enqueued as they are and each one is resolved when the
  queue loads it.
- No name search as a last resort: library, MusicBrainz links and ISRC only. Revisit if users
  report many misses.
- An external track carries its album as an `ItemMapping` built from the list answer, at no
  extra request; the album is resolved by its release id only when it is opened.
- Frontend: one generic `isExternalItem` replaces `isMusicBrainzItem` (section 8).

## Requirements

- A list costs at most one request to an external list source; a track or album costs requests
  only when the user opens, adds or plays it.
- No name searches fanned out over a list. Resolution uses the library, MusicBrainz links (our
  mirror) and the services' own ISRC/barcode lookups.
- Any external id type and media type fits the form; MusicBrainz release group (album) and
  recording (track) are implemented first, ISRC and barcode come almost for free.
- `musicbrainz://album/<release group id>` keeps resolving forever.
- Released mobile apps keep working: opening or playing such an item resolves it on the
  server, whatever the app shows for it.
- Every request goes through an existing or new throttler; a 429 is respected.

## How it works today

**Discography item.** `ArtistsController.discography` (`controllers/music/media/artists.py:650`)
lists every MusicBrainz release group of a library artist; groups not in the library become
`_album_from_release_group(...)` (`artists.py:2458-2494`): `provider="musicbrainz"`,
`item_id=<release group id>` (`:2474-2475`), the release group also as external id (`:2480`),
`provider_mappings=set()` (`:2481`), `is_playable=False` (`:2483`), and a thumb with
`provider="coverartarchive"`, `path=<release group id>` (`:2485-2493`), which
`coverartarchive.resolve_image` turns into a cover URL with one HEAD request, cached 30 days
(`providers/coverartarchive/__init__.py:86-108`). Verified.

Note on wording: the pseudo provider is on the item itself (`provider`/`item_id`), not in
`provider_mappings`, which stay empty. Its value collides with the real metadata provider's
domain: `mass.get_provider("musicbrainz")` returns the MusicBrainz metadata provider. Verified.

**Resolution.** `MusicController.get_item` special-cases the provider string
(`controllers/music/controller.py:1556-1565`): an album goes to
`AlbumsController.resolve_musicbrainz_release_group` (`controllers/music/media/albums.py:661-736`),
anything else raises "can not be resolved". The album resolver browses the group's editions on
the mirror, tries the best `_MAX_EDITION_LOOKUPS = 3` (`albums.py:112`): release details, then the
release's streaming links, then its barcode on every provider (`_album_candidates_by_barcode`,
`albums.py:1458`). Verified.

**URIs.** `create_uri` builds `{provider}://{media_type}/{item_id}` (models `helpers.py:135`);
`MediaItem` hashes and compares on `uri` (models `media_items/media_item.py:119,125`).
`parse_uri` splits `provider://type/rest` with `rest.split("/", 1)`, so an item id may contain
`:` (`helpers/uri.py:102-107`). `get_item_by_uri` is a thin wrapper on `get_item`
(`controller.py:1497-1507`). Verified.

**External ids.** `ExternalID` (models `enums.py:88-103`): `musicbrainz_artistid`, `_albumid`,
`_releasegroupid`, `_trackid`, `_recordingid`, `isrc`, `barcode`, `acoustid`, `asin`, `discogs`,
`tadb`. Library items' ids are indexed in `external_id_lookup`
(`controllers/music/database.py:557-565`, index `:691-692`). Lookups:
`get_library_items_by_external_id(s)` and `get_library_item_by_external_id(s)`
(`controllers/music/media/base.py:1087-1172`). Verified.

**get_by_external_id already exists.** Every media controller registers
`music/<type>s/get_by_external_id` (`base.py:320-324`) on `get_item_by_external_id`
(`base.py:1174-1236`): first a library item on a visible source; then, only with an id type, every
provider with `TRACK/ALBUM/ARTIST_BY_EXTERNAL_ID` (`base.py:167-171`) in turn until one answers.
The python client exposes it (`music_assistant_client/music.py`, verified by code search). The
providers only implement ISRC for tracks and barcode for albums (Spotify
`providers/spotify/provider.py:556-570`, Tidal `providers/tidal/media.py:163-180`, Deezer
`providers/deezer/media.py:114-150`, Qobuz `providers/qobuz/__init__.py:331-355`, Apple Music
`providers/apple_music/media.py:216-245`). No MusicBrainz route. Verified.

**MusicBrainz mirror.** `MB_BASE_URL = https://musicbrainz-mirror.music-assistant.io/ws/2`
(`providers/musicbrainz/api_client.py:23`), throttled at 30 requests per 10 s (`:29`), cached 30
days (`:36`). `get_recording_details` asks `inc=artists+releases+isrcs+url-rels+genres`
(`providers/musicbrainz/provider.py:437-440`): one request yields a recording's ISRCs and
streaming links. Only the MusicBrainz web service; nothing serves ListenBrainz data. Verified.

**Recording route, private to Last.fm.** `_resolve_via_musicbrainz`
(`providers/lastfm_recommendations/parsers.py:388-470`): recording details on the mirror,
`provider_mappings_from_urls` on its links, one `get_provider_item` on the first linked
provider the user has, a name/artist check. Verified.

**Top tracks.** `top_tracks` (`artists.py:772`) for a library artist calls
`get_library_artist_toptracks` (`artists.py:1258-1295`): the artist's music providers first,
metadata/plugin providers only when those return nothing. Last.fm's `get_artist_toptracks`
(`providers/lastfm_recommendations/__init__.py:237-245`) resolves each of up to 25 tracks with
`get_or_resolve_track` (`:291-313`, a search per title on the user's services, cached). The
library-only reduction is a separate change (server#7663) and is assumed to have landed.

**Playing.** `play_media` resolves every entry sequentially through `_resolve_play_media_entry`
(`controllers/player_queues/queue_loader.py:919-931, 1167-1190`), inside
`play_request_deadline` (`player_queues/helpers.py:186-206`) of `PLAY_REQUEST_RESOLVE_TIMEOUT =
60` s (`player_queues/constants.py:160`). Loading an item (`queue_loader.py:345-370`) swaps in
the library track or, in some cases, re-fetches by URI; it has no "resolve first" step.
Verified.

**Frontend.** `MUSICBRAINZ_PROVIDER = "musicbrainz"` and `isMusicBrainzItem`
(`src/plugins/api/helpers.ts:215-228`). The album page resolves such an item with
`api.getItem(ALBUM, id, "musicbrainz")` (`src/views/AlbumDetails.vue:291-294, 421-438`); the
play menu filters them out (`src/layouts/default/ItemContextMenu.vue:535, 1548`), library and
favourite actions skip them (`:772, 810`), a click opens the resolved page
(`src/helpers/media_item_actions.ts:139-143`, `src/helpers/open_musicbrainz_item.ts`). Verified.

**Mobile app.** `ServerMediaItem.MUSICBRAINZ_PROVIDER = "musicbrainz"` ("server 2.11");
`Album.isPlayable` is `!isMusicBrainzRelease`, overriding the server's `is_playable`
(`composeApp/.../items/Album.kt`); `ItemDetailsViewModel.kt` opens such an album through
`music/item`; `RefreshItemAction.kt` hides refresh for it. Verified by code search, line numbers
not taken.

**ListenBrainz.** `GET https://api.listenbrainz.org/1/popularity/top-recordings-for-artist/<mbid>`
returned 200 without a token on 2026-11-14: one unpaged array, 2170 entries and 1.47 MB for
Radiohead, each with `recording_mbid`, `recording_name`, `length`, `artists`, `release_mbid`,
`release_name`, `caa_release_mbid`, `caa_id`, `total_listen_count`, `total_user_count`. Headers
`X-RateLimit-Limit: 30`, `-Remaining`, `-Reset-In`. The same URL with any query parameter
(`?count=25`) returned 401: "Due to bad actors and AI scrapers causing undue traffic on our
sites, you need to provide an Auth token for this endpoint." Verified by request. The repository
has a ListenBrainz scrobble provider holding a user token (`providers/listenbrainz_scrobble/`).

## 1. The external item

An item known only by an external id is a normal media item with:

- `provider = "external"` (a reserved value, constant `EXTERNAL_ID_PROVIDER` in the server; no
  provider domain of that name exists, verified),
- `item_id = "<ExternalID value>:<id>"`, e.g. `musicbrainz_recordingid:9e2ad5bc-...`,
- the id also in `external_ids`, `provider_mappings` empty,
- `is_playable = True` for a track (it resolves when played), `False` for an album: a client
  opens it first. A play request that names such an album by its URI still plays, because a URI
  is resolved before anything else (`queue_loader.py:1177-1180`, verified).

URI: `external://track/musicbrainz_recordingid:<id>`, `external://album/musicbrainz_releasegroupid:<id>`,
`external://album/musicbrainz_albumid:<release id>`, `external://track/isrc:<isrc>`. `parse_uri`
handles it unchanged. No models change: the form is
a convention of `provider`/`item_id`. A helper pair in the server
(`external_item_id(type, value)` / `parse_external_item_id(item_id)`) builds and splits it;
an unknown or malformed type raises `InvalidProviderID`.

## 2. The resolver

`MediaControllerBase.get_item_by_external_id` gains a per-type route after the library step.
Order for a track by recording id:

1. Library item by `MB_RECORDING` on a visible source. DB only.
2. Recording details on the mirror (one request, cached 30 days).
3. Library item by any of its ISRCs. DB only.
4. Its streaming links (`provider_mappings_from_urls`), restricted to the user's visible sources;
   `get_provider_item` on the first one, with the name/artist check. One service request.
5. ISRC lookup (`get_track_by_external_id`) on the user's providers with
   `TRACK_BY_EXTERNAL_ID`, stopping at the first hit. At most one request per such provider.
6. Not found: `MediaNotFoundError` with a translated message. No name search.

Album by release group: the existing `resolve_musicbrainz_release_group`. Album by release id
(`musicbrainz_albumid`, what ListenBrainz names as the album of a track): the library item with
that id, then the release's details on the mirror (one request, cached 30 days), its streaming
links and its barcode, as a single edition of the release group route. ISRC track: steps 1
(by ISRC), 5. Barcode album: library by barcode, then `get_album_by_external_id`. A provider
result is swapped for its library item when there is one (existing behaviour, `base.py:1222-1227`).

Steps 2 and 4 are `_resolve_via_musicbrainz` moved from the Last.fm provider into
`TracksController` (`resolve_musicbrainz_recording`); Last.fm calls the moved function. Its
album branch (`get_release_details` and the release's links) is the release id route.

`get_item` gets a branch for `provider == "external"` that calls `get_item_by_external_id` of
the media type's controller and raises `MediaNotFoundError` on None. The `musicbrainz` branch
(`controller.py:1556-1565`) becomes an alias that maps to the release group route.

Caching: the resolved item's URI is cached per external id for 7 days (`use_cache`, keyed with
the user's visible source set so a household member never gets another member's account). A
cached URI that no longer resolves is dropped and the chain runs again.

## 3. Top tracks from ListenBrainz

A small client in a helper of its own, so the list source can later move to our mirror or get
a token without touching its callers, with its own `ThrottlerManager(rate_limit=1, period=1)`,
honouring 429 and `X-RateLimit-Reset-In`, caches the parsed and capped list per artist MBID for
7 days. A 401 is treated as "no list". Only the first `TOP_TRACKS_LIMIT = 50` entries are kept
after de-duplication by normalised title (first, i.e. most played, wins).

`get_library_artist_toptracks` uses it as the metadata fallback for a library artist with an
MBID, ahead of Last.fm: each entry is matched to a library track by recording id (one DB query
per entry or one batched `IN` query); the rest become external tracks with name, artists (the
library artist as `ItemMapping` where the MBID matches), duration, an album `ItemMapping`
(`external://album/musicbrainz_albumid:<release_mbid>`, named `release_name`; built from the
list answer, no request) and a cover from `caa_release_mbid`/`caa_id` as a direct Cover Art
Archive URL (no lookup). Last.fm stays for
artists without an MBID, matching the library only.

## 4. Playback

External tracks are enqueued as they are. `_resolve_play_media_entry`
does not resolve an external track; the load step (`queue_loader.py:345`) resolves it through
`get_item_by_uri` before `get_stream_details`, replaces `queue_item.media_item` and keeps the
origin. A failure marks the item unavailable and the queue moves on, as for any unavailable
item. Playing one external track resolves at once (it is the item to load).

The queue persists items across restarts (assumption: the external URI then resolves on load,
which this design already covers).

## 5. Request budget per user action

| action | list source | mirror | music services |
|---|---|---|---|
| open artist page, top tracks row | 1 ListenBrainz per artist per 7 days | 0 | 0 |
| open or play one external track, in library | 0 | 0 | 0 |
| same, not in library | 0 | 1 (cached 30 d) | 1 by link, else at most 1 per provider with ISRC lookup |
| open the album of an external track | 0 | 1 release lookup (cached 30 d) | 1 by link, else the barcode lookup |
| play the whole list of 50 | 0 | per track when loaded | per track when loaded, only tracks reached |
| open a discography album | unchanged: 1 browse + up to 3 release lookups on the mirror; 1 service request per edition link, barcode fan-out only when no link works |

Resolving the whole list when it is played was rejected: a request per track of a list for one
action is what the usage policy rules out. Resolving on load costs the same as the user pressing
play on each track.

## 6. Existing `musicbrainz` items and stored references

No server path stores such an item: they are not playable (the queue never holds one),
favouriting resolves the URI and adds the result to the library first (`set_item_favorite`,
`controller.py:1656-1665`), adding to the library resolves first (`controller.py:1796`). Stored references are in clients only: browser history and links of the
frontend (`/album/musicbrainz/<id>`), mobile app state, and URIs a user may have saved in Home
Assistant scripts (assumption, unverifiable). Therefore `musicbrainz://album/<id>` stays a
permanent alias; no migration.

## 7. API compatibility

- No new command, no new parameter, no model field. `get_by_external_id` gets better answers.
  Per `constants.py:65-73` no `API_SCHEMA_VERSION` bump is needed for any step.
- The discography switch changes what `provider` holds for those albums: by the constant's own
  rule no bump. The released mobile app keys on `"musicbrainz"`, so until it is updated it shows
  these albums as ordinary, playable albums. Playing one sends its URI, which the server resolves
  at that moment (one album, one explicit action), and opening one goes through `get`, which
  step 1 routes to the resolver. Both keep working (assumption about what the app sends, to be
  confirmed with the mobile app team).
- External tracks in top tracks reach the mobile app once the ListenBrainz step lands. The app would show them as
  normal playable tracks (its `isPlayable` override covers albums only, assumption from the code
  search): playing sends the URI, which the server resolves, so it works; opening a track page
  calls `music/tracks/get` with provider `external`, which `MediaControllerBase.get`
  (`base.py:790-825`) does not route to `get_item`. Step 1 adds that routing in `get`, so old
  apps open them too.
- Home Assistant: no discography or top tracks use found in the integration (code search on
  `home-assistant/core`); playing an external URI from a script works.

## 8. Frontend

One generic `isExternalItem` (`provider === "external"`) replaces `isMusicBrainzItem`. Every
call site asks the same thing (has this item been found yet) and none of them is about
MusicBrainz: `ListviewItem.vue`, `EditorialMediaCard.vue`, `ReleaseShelf.vue`,
`ItemsListing.vue`, `ItemContextMenu.vue` and `helpers/media_item_actions.ts`. The helper does
not need to know the `musicbrainz` value: the server switches the discography in the same
release and the frontend always ships with its server. An old link (`/album/musicbrainz/<id>`)
still opens, because the server keeps resolving that address.

Whether such an item can be played comes from the server's `is_playable` (false for an album,
true for a track), not from the helper, so the extra `!isMusicBrainzItem` in the play menu
goes. External albums take the existing discography path (`AlbumDetails.vue:421-438`
generalised to `resolveExternalItem`, which passes the item's own provider and id). External
tracks open via `openMusicBrainzItem` (renamed `openExternalItem`), get the muted styling, and
have no favourite, playlist or library actions until resolved (an "add to library" resolves
first and is allowed, as for albums today).

## 9. What stays

Library schema, `external_id_lookup`, provider contracts (`get_*_by_external_id`), the
discography listing and its cost, Last.fm for artists without an MBID, the throttlers, the
readrate pacing and every other usage-policy guard. `MIN_SCHEMA_VERSION` and
`API_SCHEMA_VERSION` are untouched.

## Testing

- Resolver: library hit by recording id and by ISRC makes no requests; link hit makes one
  service request; ISRC fan-out stops at the first hit; not-found raises with a translation key;
  a hidden source is never used.
- `get_item("external", ...)`, `get("external", ...)` and `musicbrainz://album/...` alias.
- Last.fm still resolves through the moved recording resolver (existing tests keep passing
  unchanged; a failing one is a regression, not a test to adjust).
- ListenBrainz client on a recorded response: cap, de-duplication, cache hit, 429 handling,
  401 handled as "no list".
- Top tracks: library matches replace external entries; order kept.
- Album by release id: a library hit makes no request; otherwise one release lookup on the
  mirror and one service request.
- Queue: playing 50 external tracks makes no resolver call up front; load resolves exactly the
  loaded item; an unresolvable item is skipped.

## Pull requests

| # | issue | repo | title | depends on | size |
|---|---|---|---|---|---|
| 1 | #453 | server | Resolve items by external id on demand (resolver incl. release ids, `external://`, Last.fm reuse) | - | medium |
| 2 | #454 | frontend | Show, open and play external items | 1 | small |
| 3 | #458 | server | Discography lists external albums | 1, released together with 2 | tiny |
| 4 | #455 | server | Top tracks from ListenBrainz | 1, 2 | medium |
| 5 | #456 | server | Resolve external tracks when the queue loads them | 1 | small |
| 6 | #457 | mobile-app | Understand external items | 1 | small (other team) |

## Still to find out

- Whether MetaBrainz will give the project a token for the ListenBrainz popularity endpoint, and
  under which conditions.
- Whether our own mirror can serve or cache the popularity data, so the list does not depend on
  a public endpoint.
- With the mobile app team: what a released app does when it plays or opens an album in the
  external form (section 7 assumes it sends the URI).

## Later

- Remove the app's `musicbrainz` handling once every supported server sends `external`.
- External items for other lists: recommendations, charts, imported playlists by ISRC, similar
  tracks from open data.
- Lazy resolution for an external album in the queue (play an unresolved album directly).
