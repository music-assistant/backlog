# Stream source selection: technical plan

Status: proposal 2026-10-09. Owner: Marcel van der Veldt. This document is the technical
companion of the "Play albums from one source and choose which source plays" epic (#260) on
the project board. It is written so it can be fed to an agent for implementation; every claim marked
"verified" was checked in the referenced code (server at `30d70995f`, `music-assistant-models`
1.1.217, frontend `main` of 2026-10-09).

## Summary

When a library track is available from more than one source, Music Assistant (MA) picks the
copy to play purely by audio quality, per queue item. Playing an album can therefore serve one
track from a compilation copy, another from a remaster and a third from a streaming service,
which breaks gapless playback, album loudness and the mastering flow. Nobody can say "my own
files first", upgrade a low-quality rip to a hi-res streaming copy, or keep a remaster apart
from the original, and which copy plays is invisible.

The fix has three parts. The queue item records where it was played from (album, playlist,
folder, podcast, audiobook) and the exact copy that listing had for it; for an album one source
is chosen up front, so every track is pinned to the same place. A small, pure ranking module
replaces the quality-only sort in the streams controller: pin first, then "local first" when
asked, then the playback user's own accounts, then quality, then a fixed tie-break. Two
settings control it: a source selector (best quality, prefer local, best quality per track) and
an upgrade threshold. An opt-in library option keeps remastered versions apart. No stored data
changes; the album-to-copy link is captured at enqueue time from listings the album page already
fetches.

## Decisions

Taken with the maintainer on 2026-10-09:

- The container pin outranks the "own account first" ordering. The hard rule stays: another
  account of a streaming service the user owns never serves them.
- The album-to-copy link is captured at enqueue time only. Persisting it in the library
  database is a later follow-up, once there is a consumer ("play this version", a safe split of
  merged remasters).
- An opt-in "keep remastered versions apart" matching option is part of this epic but is the
  lowest-priority item: once album playback uses the album's own copies, the merged library
  track is no longer a playback problem.
- Two settings, nothing more: one source selector and one upgrade threshold. A user-ordered
  provider list and a priority strictness setting are left out; a custom order goes to "Later"
  if the votes ask for it.
- Container-aware playback is on by default. A single track played by provider URI is not
  pinned in this version (containers only).
- Open PR server#6683 (folder-affinity tie-break) is superseded by this epic.

## Requirements

- An album, playlist or folder plays every track from the source it was started from, falling
  back to another copy only when that one is unavailable or at its stream limit.
- A playlist from a streaming service plays from that service's instance.
- A filesystem folder plays the files in that folder, also when the library track has copies
  elsewhere.
- Podcast episodes and audiobooks play from the provider they were started from, so progress
  syncs there.
- The user can prefer local and self-hosted sources over streaming services, and can let a
  copy below a chosen quality be replaced by a better one.
- Ties are deterministic across restarts.
- Which copy plays, and why, is visible in the log and in the queue.
- The user's access rules, the stream-limit handling, track linking and the album tracks API
  are unchanged. Nothing becomes a way to download or keep provider audio.

## How it works today

Verified in the server at `30d70995f` unless noted.

**Enqueue.** `play_media` resolves every requested item through `get_item_by_uri`, which
prefers the library item (`controllers/music/media/base.py:724-739`), expands containers in
`media_resolver._resolve_media_items` (`controllers/player_queues/media_resolver.py:713-828`)
and concatenates the results (`controllers/player_queues/queue_loader.py:964-976`). The link
between a container and its tracks is dropped there. Every queue item is created by
`build_queue_item` (`controllers/player_queues/helpers.py:144-161`; other callers:
`queue_loader.py:594, 682, 737, 1092`, `playback_tracker.py:550`, the party and ai_radio
providers). `QueueItem` has no origin field, only `extra_attributes` for primitives
(`music_assistant_models/queue_item.py:24-43`). Queue-level context exists (`source_items`,
`enqueued_media_items` of at most 10, `PlayerQueue.sources`; `player_queues/state.py:50-59`),
is cleared on PLAY and REPLACE and never records a folder.

**Load.** `_load_item` (`queue_loader.py:305-391`) swaps the queued track for the full library
track, "as there might be other qualities available" (`:331-340`), restores the album
(`:357-368`) and asks for stream details (`:386-391`).

**Selection.** `_get_stream_details` (`controllers/streams/audio.py:3389-3553`) reads
`playback_sources` (`helpers/provider_access.py:229-244`): `allowed` is the user's visible
sources minus the other accounts of streaming services the user owns, `preferred` is the
enabled sources the user owns. `_get_streamdetail_candidates` (`audio.py:3761-3800`) sorts the
mappings by `mapping.quality` descending, expands each through `_get_mapping_providers`
(`:3818-3860`: the mapped instance, then sibling accounts of the same streaming catalog, all
filtered on availability and `allowed`), and puts owned instances first. Ownership beats
cross-service quality (pinned by
`tests/controllers/streams/test_streamdetails_provider_attempts.py:318-338`).
`_request_streamdetails` (`:3980-4020`) tries the candidates in order; a `MediaNotFoundError`
on the mapping's own instance marks the mapping unavailable.

**Score.** `ProviderMapping.quality = audio_format.quality + priority`
(`music_assistant_models/media_items/provider_mapping.py:39-61`): `priority` is +1 when in the
library, +2 for the domains `filesystem_local`/`filesystem_smb`/`filesystem_nfs` (the last two no
longer exist, `controllers/config/filesystem_consolidation.py`), +1 for any other non-streaming
instance. `AudioFormat.quality` (`audio_format.py:44-57`) is `sample_rate_kHz + bit_depth` for
lossless content types and about 1 to 3 for lossy; it looks at `content_type` only, so ALAC in
an m4a container scores lossy, and it divides by `channels`.

| copy | score |
|---|---|
| local FLAC 16/44.1 | 63 |
| Tidal FLAC 16/44.1 | 60 |
| Qobuz FLAC 24/96 | 120 |
| local MP3 320 | 4 |
| Spotify OGG 320 | 2 |
| unknown format (podcast, MusicBrainz link) | 1 |

"Prefer local" is at most a +3 tie-break. Equal scores keep the iteration order of a `set`,
whose hash seed changes per process, so the pick can differ after a restart.

**Selection moments.** The current item in `_load_item`; the next item near the start of the
current one (`player_queues/stream_feeder.py:331-340`, `controller.py:1395-1461`); again after
the 600 s expiry (`music_assistant_models/streamdetails.py:184-190`); on seek through
`play_index`; on a stream-limit error with the busy instance excluded (`audio.py:3555-3735`).
When every copy is at its limit, `_discover_alternative_provider_mappings` (`:3896-3978`) runs
`tracks.match_provider(strict=True)` on the other streaming services and stores the result with
`add_provider_mappings` (`:3961-3967`), which merges library rows on a conflict.

**Album tracks.** `albums.tracks()` (`controllers/music/media/albums.py:399-498`) returns the
library rows with all their mappings, fetches every album mapping's listing and lets
`select_album_tracks` (`media/album_tracks.py:20-85`) correlate provider entries with library
rows by provider id, ISRC, position or unique title. Matched entries are dropped; the
correspondence is computed and thrown away. The filesystem provider lists a mapped album as the
library rows that have any mapping on the instance (`providers/filesystem_local/__init__.py:
2016-2020`), so with the 1969 and the 2009 rip on the same instance nothing says which file
belongs to which album; `_entries` skips those rows (`album_tracks.py:207-209`). The
`provider_mappings` table has no album column and `album_tracks` links library ids only
(`controllers/music/database.py:475-504`).

**Other containers.** Provider playlists return provider items with one mapping
(`controllers/music/media/playlists.py:189-238`); filesystem m3u entries are library rows
relabelled with the instance (`filesystem_local/__init__.py:2493-2520`); a builtin (MA)
playlist entry carries the alphabetically first of its stored mappings as its identity
(`helpers/playlists.py:520-538`). Folder browsing returns `ItemMapping` entries with the
relative path as id and replaces in-library files by the library track
(`filesystem_local/__init__.py:409-517`, `:500-516`); a track mapping's `item_id` is the
relative file path (`:2553-2575`), an album's is the folder (`:3441-3464`).

**Same-album knowledge.** The crossfade guard compares the two items' albums with
`compare_item_ids` (`audio.py:3039-3110`); album loudness matches the album against
`enqueued_media_items` (`queue_loader.py:419-438`). `audio_analysis.py:653` and
`player_queues/smart_fade_ordering.py:276` guess the played copy by sorting on quality.

**Matching.** `compare_track` (`helpers/compare.py:385-503`) accepts an equal MusicBrainz
recording id before any version or album check (`:407-416`), so an original and its remaster
merge into one library track while album matching keeps the albums apart (edition tokens and
tracklist fingerprints, `compare_album_evidence` `:217-248`). The recording-conflict tokens
(`:59-68`) are acoustic, cover, demo, instrumental, karaoke, live, remix and session; "remaster"
is absent, `_is_missing_remaster_version` (`:1367-1376`) even forgives a missing remaster
version in `compare_track_evidence` (`:579-685`). A library track's album comes from the
`track_album` JSON subquery (`media/tracks.py:168-183`), which carries no version. Local files
store the recording id (`filesystem_local/__init__.py:2654-2655`); the release-track id
(`ExternalID.MB_TRACK`) is only stored for CUE tracks (`filesystem_local/cue.py:526-528`),
although `helpers/tags.py` already parses it (`:1339-1340`, with a per-container ambiguity at
`:1034` and `:1190`).

**Config.** Core entries live in `get_config_entries` (`controllers/streams/controller.py:
422-514`), their strings in `controllers/streams/strings.json`, category labels in
`music_assistant/strings.json` under `config_categories`. The per-queue "global" pattern is
`controllers/player_queues/config.py:158-188`. The frontend renders `multi_value` entries as a
multi select in click order (`frontend/src/views/settings/ConfigEntryField.vue:197-215`).

## 1. Queue item origin

`music-assistant-models` (`queue_item.py`), one release and a pin bump in the server's
`pyproject.toml:29`:

```python
@dataclass(kw_only=True)
class QueueItemOrigin(DataClassDictMixin):
    """Where a queue item was played from, and the copy that listing had for it."""

    container: ItemMapping | None = None  # album, playlist, podcast, audiobook or folder
    provider_instance: str | None = None  # the listing's mapping for this item (the pin)
    item_id: str | None = None


class QueueItem:
    origin: QueueItemOrigin | None = None  # new optional field, after `available`
```

A typed field rather than `extra_attributes`: it is structural data the frontend shows and the
ranking reads. `(provider_instance, item_id)` is exactly the candidate identity the ranking
uses (`audio.py:3788`), so no `ProviderMapping` copy is stored and nothing drifts when the
library mapping's format is updated later. `to_cache`/`from_cache` need no change; old caches
deserialize with `origin=None`; clients ignore the extra key.

Same models release, separate tiny fixes: `AudioFormat.quality` treats a copy as lossless when
`content_type` or `codec_type` is lossless (ALAC in m4a) and guards `channels == 0` (the guard at
`queue_loader.py:980-981` exists for this); `ProviderMapping.priority` drops the dead
`filesystem_smb`/`filesystem_nfs` domains.

## 2. Origin capture at enqueue time

Server, `controllers/player_queues/`.

- `media_resolver._resolve_media_items` returns `ResolvedItem(item, origin)`; branches without
  a container wrap the item with `origin=None`. `build_queue_item(queue_id, media_item,
  origin=None)` assigns it after `QueueItem.from_media_item`. `_handle_play_media` passes it
  through (`queue_loader.py:1027-1032`). The pool and autoplay builders
  (`queue_loader.py:594, 1092`, `playback_tracker.py:550`) pass the dynamic source's origin
  where the source is known, otherwise `None`; the party and ai_radio providers are unchanged.
- Pin rule, `_origin_for(container, item)`: the pin is `(item.provider, item.item_id)` iff the
  item is a provider item (`item.provider != "library"`) and the container is not a builtin (MA)
  playlist, whose entries carry an arbitrary provider identity. The container is always
  recorded. This covers provider playlists, library playlists resolved to one instance
  (`playlists.py:206-208`), filesystem m3u playlists, podcast episodes and audiobooks (gPodder
  versus Spotify) and the tracks of a dynamic source.
- Folder play (`media_resolver._get_folder_items`, `:830-868`): the container is a hand-built
  `ItemMapping(media_type=MediaType.FOLDER, item_id=folder.item_id, provider=folder.provider,
  name=folder.name)` (`ItemMapping.from_item` reads metadata a `BrowseFolder` lacks). The pin is
  the child's own identity for an `ItemMapping` or provider item; for a library track (the
  filesystem `browse` substitutes those) it is the track's mapping on `folder.provider` whose
  id starts with `folder.item_id + "/"`, else the lowest id on that instance. Children of
  subfolders keep the folder container. When the folder is an album folder (its path is the
  filesystem album's provider item id, `filesystem_local/__init__.py:3441-3446`) or a disc
  subfolder of one (`is_disc_dir` in `filesystem_local/helpers.py`, the parent folder), the
  container is that library album's `ItemMapping` instead; the pins are still the folder's
  files. `_plays_as_album_track` (`queue_loader.py:419-438`) then also returns True when
  `origin.container` is an album, so an album folder plays with album loudness like the album
  itself; the loudness value is a per-file measurement of the mapping that streams
  (`audio.py:1573-1584`), so no album lookup is needed. `enqueued_media_items` stays as it is, so
  autoplay seeding and album play credit are unchanged.
- Album play: section 3 chooses one source instance and pins each track to that instance's
  listed entry.
- A single track played by URI (library or provider) gets no origin in this version.
- `_load_item` leaves `origin` alone when it swaps in the library track; it logs the origin at
  debug level next to the selected stream.

## 3. Album playback from one source

- `media/album_tracks.py`: `AlbumListing(tracks: list[Track], listed_ids: dict[str, dict[str,
  str]])`, keyed by instance id, then by track uri, valued with the provider item id that
  instance lists for the row. A sibling of `select_album_tracks` returns the correspondence
  (provider id, then ISRC, then position or unique title, the same evidence the album page
  trusts) instead of dropping it; `select_album_tracks` itself is unchanged. A provider-only
  entry maps to its own id.
- `media/albums.py`: `tracks()` is split into `_assemble(...) -> AlbumListing` and the existing
  `tracks()` returning `.tracks`, so the API output is unchanged. New
  `playback_listing(item_id, provider_instance_id_or_domain, in_library_only=False)` for the
  resolver. The instance key is the instance that served the listing (`albums.py:443-449`);
  with `in_library_only` there are no listings and the rows' own mappings per instance are
  used, lowest id first.
- `providers/filesystem_local/__init__.py`, `_iter_album_tracks` (`:2016-2020`): for a mapped
  album, yield copies of the library rows (`dataclasses.replace`) with their mappings narrowed
  to this instance's files under the album folder: `dirname(path) == folder` or
  `dirname(dirname(path)) == folder`, mirroring `_scan_folder_tracks` (`:2032-2060`). A
  synthetic `artist/album` id has no folder; then all own mappings stay, lowest id first. Copies,
  not mutation: `get_album_tracks` (`:952-962`) returns the rows verbatim for an album that is
  not in the library.
- `select_container_source(listed, preferred, policy)` in the ranking module orders the
  listing instances by: non-streaming first when the mode is prefer local; the playback user's
  own accounts; a complete listing (every library row has an entry) before a partial one;
  quality (lowest tier over the listing, then the median score); non-streaming; instance id. The
  resolver pins each track to the chosen instance's entry. A track the chosen instance does not
  list keeps `origin.container` with an empty pin and falls to the generic order; the resolver
  logs that at debug level. In "best quality per track" mode no source is chosen and nothing is
  pinned; the container is still recorded.
- An album that is not in the library returns provider items (`albums.py:406-417`); their own
  identity is the pin. `_load_item` may still swap in a library row for some of them; the pin
  is honoured all the same.

## 4. Ranking module

New `controllers/streams/stream_sources.py`, pure and synchronous, no provider calls.

- `QualityTier(IntEnum)`: `UNKNOWN = 0`, `LOSSY_LOW = 1`, `LOSSY_HIGH = 2` (256 kbps and up),
  `LOSSLESS = 3`, `HIRES = 4` (above 48 kHz or above 16 bit). `quality_tier(audio_format)`.
- `StreamSourcePolicy` (frozen dataclass: `mode`, `upgrade_below: QualityTier | None`) and
  `read_stream_source_policy(mass, queue_id=None)`; the `queue_id` parameter exists so a
  per-queue override can be added later without touching callers.
- `SourceCandidate(mapping, provider, is_streaming, reason)`.
- `rank_stream_sources(candidates, origin, preferred, policy) -> list[SourceCandidate]`. The hard
  filters stay where they are: `_get_mapping_providers` drops unavailable mappings, disallowed
  and excluded instances and adds sibling accounts of a streaming catalog. The sort key:

```
tier(c)   = quality_tier(c.mapping.audio_format)
score(c)  = c.mapping.quality                       # format score plus today's local/in-library bonus
pinned(c) = origin is not None and (c.provider.instance_id, c.mapping.item_id)
                                 == (origin.provider_instance, origin.item_id)

key(c) = (
    not pinned(c)                    if mode != best_quality_per_track else 0,   # 1. the pin
    c.is_streaming                   if mode == prefer_local            else 0,   # 2. local first
    c.provider.instance_id not in preferred,                                       # 3. own accounts
    -tier(c), -score(c),                                                           # 4. quality
    c.provider.instance_id != c.mapping.provider_instance,                         # 5. mapped instance first
    c.provider.instance_id, c.mapping.item_id,                                     # 6. deterministic
)
```

  Two passes when an upgrade threshold is set (section 6): the first pass finds the would-be
  winner; when its tier is below the threshold, the candidates with a higher tier move to the
  front, ordered among themselves by the same key.
- Reasons, one short string per candidate for the debug log and a later UI: "pinned to
  <container>", "local source", "own account", "hi-res 96/24", "upgrade: lossless over lossy",
  "tie-break".
- `_get_streamdetail_candidates` (`audio.py:3761-3800`) becomes a wrapper that expands the
  mappings as today, calls `rank_stream_sources` with `queue_item.origin` and returns
  `[(mapping, provider)]`, so `_request_streamdetails` (`:3980`) is untouched. The
  blocked-for-user rebuild (`:3453-3464`) passes the same origin.
- `audio_analysis.py:653` and `smart_fade_ordering.py:276` use the shared sort key; where a
  queue item with stream details is at hand they read `streamdetails.provider` and `item_id`
  instead of guessing.
- Determinism: the inputs (the item's `origin`, its library mappings, the user's allowed and
  preferred sources, the policy, the excluded instances) are the same at every selection
  moment, so the current item, the preloaded next item, a reselection after the 600 s expiry, a
  seek and a reselection after a stream-limit error all agree. Set iteration never reaches the
  sort.

## 5. Settings

Core streams config, new category `source_selection` ("Source selection"), two entries:

| key | type | options (default first) | meaning |
|---|---|---|---|
| `stream_source_mode` | STRING | `best_quality`, `prefer_local`, `best_quality_per_track` | Which copy plays when a track is available from several. **Best quality**: the highest quality; an album, playlist or folder still plays from the source you started it from. **Prefer local**: your local and self-hosted sources (files, Plex, Subsonic, ...) before streaming services, then the highest quality; albums, playlists and folders play from the source you started them from. **Best quality for every track**: every track on its own, even inside an album (previous behaviour). |
| `upgrade_below_quality` | STRING | `off`, `lossy_high`, `lossless`, `hires` | When the copy chosen above is below this quality, play a better one if the track has it, and search your other streaming services once for a match. |

"Local" means a non-streaming music source (`MusicProvider.is_streaming_provider` is False,
the set the `non_streaming_providers` cache holds, `mass.py:1612-1634`), which also settles the
NFS and OpenSubsonic misclassification complaints. The upgrade threshold is the only relaxation
of the pin and of prefer-local, so "local first unless it is a poor MP3" needs no tolerance
setting.

Strings in `controllers/streams/strings.json` (`config_entries.<key>.label`, `.description`,
`.options`, `.option_descriptions`) and the category label in `music_assistant/strings.json`.
Global in this version; per-queue overrides can follow the global/per-queue pattern later.
The defaults reproduce today's ordering, with two visible differences: container plays become
consistent and ties are deterministic.

## 6. Upgrade below a quality threshold

In `_get_stream_details` (`audio.py:3389-3553`), after ranking: when `policy.upgrade_below` is
set, the item is a library track and the would-be winner's tier is below the threshold, run the
existing `_discover_alternative_provider_mappings` (`:3896-3978`) once per queue item (a guard
keyed like `_stream_details_locks`), bounded by `STREAM_SLOT_MATCH_TIMEOUT`, then rank again.
The log line in that function stops assuming a stream limit (it becomes a parameter). That
path must store its matches with `add_unclaimed_provider_mappings` (`media/base.py:1337-1352`)
instead of `add_provider_mappings` so a playback-time match never merges library rows; this
applies to the existing stream-limit use as well. A copy the search does not find stays as
chosen and is logged; the next-track preparation keeps passing `allow_provider_match=False`
(`stream_feeder.py:108-117`), so the search runs on the current item only.

## 7. Keep remastered versions apart

Lowest priority, independent of sections 1 to 6; it can move to "Later" without affecting the
rest. Once sections 2 to 4 are in, playing the 2009 album plays the 2009 files and playing the
1969 album plays the 1969 files, so this option only serves users who want both versions as
separate tracks and albums in the library.

Music controller config entry (`controllers/music/controller.py:334-358` area), `advanced`,
`immediate_apply`, off by default. When on:

- `compare_track` (`compare.py:385-503`): a remaster-version conflict (one side has a remaster
  token and the other none, or the remaster years differ) returns False before the
  recording-id, AcoustID and secondary-id acceptance; an equal release-track id (`MB_TRACK`)
  still matches. The version compared is the track's own, falling back to its album's version.
- `compare_track_evidence` (`:579-685`): "remaster" joins the conflict tokens
  (`_track_versions_conflict`, `:1300-1306`) and the `_is_missing_remaster_version` leniency is
  off.
- `_compare_album_version` (`:1077-1098`): a remaster token next to a blank version is
  `NO_MATCH` instead of `INSUFFICIENT`, so a remastered album from a streaming service is not
  merged into the local original through the tracklist fingerprint.
- The compare helpers are pure. The music controller sets a module-level matching option once
  at setup and whenever the entry changes; threading an explicit parameter is acceptable if it
  turns out cleaner.
- Library tracks carry their album's version: `'version', albums.version` joins the
  `track_album` JSON subquery (`media/tracks.py:168-183`); `ItemMapping` already has the field.
  No schema change.
- Hygiene in the same issue: local files store the MusicBrainz release-track id as
  `ExternalID.MB_TRACK` (today only CUE tracks do). `helpers/tags.py` keeps the id under
  `musicbrainztrackid`, which is a recording id for ID3 and MP4 and a release-track id for
  Vorbis and APE, so the property is disambiguated per tag format. External ids are
  append-only; no migration.
- Already merged tracks and albums stay merged; the duplicate-repair job
  (`controllers/music/controller.py:3311-3367`) already refuses to merge differing versions.
  The docs say so. A safe split needs the persisted album-to-copy link (later).
- Streaming matches found by ISRC (`media/tracks.py:977-1021`) and MusicBrainz URL links
  (`media/base.py:1354-1390`) go through the same comparison, so with the option on they only
  attach when the versions agree.

## 8. Frontend

`frontend` repo: `origin?: QueueItemOrigin` on the `QueueItem` interface
(`src/plugins/api/interfaces.ts`); "Playing from <source>" in the queue item details and the
now-playing view, from `streamdetails.provider` and `origin.container`; translations for the two
entries and the category. No new config widget.

## 9. What stays

Track linking defaults and the usage-policy guards; `allowed` (never another account of a
service the user owns); the `_request_streamdetails` fallback loop and the unavailable-mapping
marking; stream-limit handling; the `albums.tracks()` output; the same-album crossfade guard
(switching it to `origin.container` is a follow-up; album loudness gains the origin rule in
section 2); the search and browse ordering; provider quality settings.

## Worked examples

Default settings unless stated; "fs" is a filesystem instance nobody owns.

1. **Library album, fs 1969 rip, fs 2009 remaster rip, Tidal 24/96.** The 1969 and the 2009
   album are separate library albums. Playing the 1969 album: the fs listing is narrowed to the
   1969 folder (complete, lossless), Tidal is complete and hi-res, so Tidal is the album source
   and every track is pinned to its Tidal entry; the 2009 files are never candidates for this
   play. With prefer local the fs instance is the source and every track plays the 1969 file.
   Playing the 2009 album plays the 2009 files or its own streaming copy, never the 1969 ones.
2. **Tidal playlist, one track also local as FLAC 24/96.** The items are Tidal provider items,
   so each is pinned to Tidal; `_load_item` swaps in the library track; the ranking puts the
   Tidal copy first ("pinned to <playlist>"), the local file second. With the upgrade threshold
   at hi-res, the Tidal copy (lossless) is below it and the local hi-res file plays.
3. **Local MP3 album, Qobuz hi-res not linked yet, threshold lossless.** The album source is
   fs (the only listing), every track is pinned to its MP3. At play time the winner is
   high-bitrate lossy, below lossless; no better copy is linked, so one bounded strict match on
   Qobuz runs, the unclaimed mapping is stored, and the Qobuz copy jumps the pin. Tracks Qobuz
   does not have stay MP3 and are logged.
4. **Single track from search.** No origin: own accounts, then quality tier and score, then
   local, then in-library, then ids. The same for a provider URI in this version.
5. **Podcast episode on gPodder and Spotify.** Episodes resolved from gPodder are pinned to
   gPodder; without the pin both copies are tier unknown and only the id tie-break would decide.
6. **Stream-limit error on the pinned instance.** `_get_audio_buffer` excludes the busy
   instance, the next candidate plays ("at stream limit" logged); the final blocking pass clears
   the exclusion and the pin is first again. Same inputs, same order.
7. **A household member who owns Spotify plays the admin's shared local album.** The pin (fs)
   comes before the own-account preference, so the local files play. For an artist play
   without a container, the member's own Spotify still comes first, as today, unless the mode is
   prefer local.

## Testing

- New `tests/controllers/streams/test_stream_sources.py`: table-driven ranking for every key
  level; the three modes; the upgrade jump over the pin and over prefer-local; the two-pass
  behaviour; `select_container_source` with complete and partial listings; `quality_tier` for
  ALAC in m4a, unknown bitrate and zero channels; the policy reader.
- New `tests/controllers/music/test_album_playback_listing.py`: correspondence for a row with
  two same-instance mappings, a provider-only bonus track, an unavailable mapped instance served
  by a sibling account. Filesystem `_iter_album_tracks` narrowing (folder, disc subfolder,
  synthetic id fallback).
- New `tests/controllers/player_queues/test_media_resolver_origin.py`: Tidal playlist pin,
  builtin playlist without pin, folder pin for an `ItemMapping` child, a provider item and a
  library track with two fs mappings, library URI without origin.
- Extend `tests/controllers/streams/test_streamdetails_provider_attempts.py` (the pin precedes
  quality and the owned account; the pin is ignored in best quality per track; the pin is
  skipped when excluded; ties resolve by instance and id; the existing steering tests stay),
  `test_stream_capacity*.py` (a pinned instance at its limit falls back and returns),
  `tests/controllers/player_queues/test_folder_playback.py` and `test_prefer_album_loudness.py`
  (the origin survives `_load_item`), `tests/helpers/test_compare.py` (the remaster option on
  and off).
- Manual: a library album present as two local folders and on Tidal; play the 1969 album, the
  2009 album, a Tidal playlist and a folder; read the reasons in the debug log and the "Playing
  from" line; switch the modes and the threshold; enable the remaster option and rescan a
  remaster folder.
- `pre-commit run --all-files` and `pytest -n auto --dist loadfile` green per PR.

## Sub-issues

One PR each, in this order; 7 is independent of the others.

| # | issue | sections | size |
|---|---|---|---|
| 1 | #261 Models: queue item origin and quality score fixes | 1 | tiny |
| 2 | #262 Server: one deterministic ranking for stream sources | 4 | medium |
| 3 | #263 Server: record where a queue item was played from | 2 | medium |
| 4 | #264 Server: play an album from one source | 3 | large |
| 5 | #265 Server: source selection settings | 5 | small |
| 6 | #266 Server: upgrade low-quality sources at playback time | 6 | small |
| 7 | #267 Server: keep remastered versions apart (opt-in) | 7 | medium |
| 8 | #268 Frontend: show which source is playing | 8 | small |
| 9 | #269 Docs: source selection settings and the remaster option | 4 to 7 | small |

## Later

- A persisted album-to-copy link (a nullable `album_provider_item_id` on `provider_mappings`,
  schema 66, dev only), with a "which version is playing / play this version" action in the
  UI, a correct per-album default album and a safe split of merged remasters.
- A user-ordered list of providers, if the votes ask for it.
- Per-queue overrides of the two settings.
- The same-album crossfade guard and album loudness driven by `origin.container`.
- Closing server#6683 as superseded, with credit for the two test scenarios it ported.

## Changes during implementation

- 2026-10-09, #262 (server#6804): the local and in-library bonus keeps today's weight. Within a
  quality tier the ranking uses `ProviderMapping.quality` (format score plus the +2/+1 bonus)
  rather than the raw format score, so a local MP3 320 still beats a streaming OGG 320 and a
  file on disk still beats the same file in Plex; the separate "non-streaming, in-library"
  tie-break keys are gone. Prefer-local (#265) adds its preference on top.
- 2026-10-09, #262: one extra key, the mapped instance ranks before a sibling account of the
  same service standing in for it (keeps today's order instead of alphabetical instance ids).
- 2026-10-09, #262: a streaming account that maps the item itself is ranked on, and marked
  unavailable through, its own mapping instead of a higher-quality sibling's mapping.
- 2026-10-10, #263: a folder play of an album folder (or a disc subfolder) records the album as
  the origin container and gets album loudness through `_plays_as_album_track`; folds in the
  older "give folder plays an album for loudness" task without touching
  `enqueued_media_items`.
