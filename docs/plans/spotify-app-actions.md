# Follow the Spotify app on the Music Assistant device: technical plan

Status: decided with the maintainer on 2026-11-08. Technical companion of the epic "Follow what
the Spotify app starts on the Music Assistant device" (#391). Line numbers refer to server
`origin/dev` at `92e89173f`. `soloist.py` is `music_assistant/providers/spotify/backends/soloist.py`,
`runtime.py` is `music_assistant/providers/spotify_connect/soloist/runtime.py`.

## Summary

When Music Assistant plays Spotify through the Soloist engine, the Spotify app shows a device
called "Music Assistant Playback". What people do in the app on that device is lost today, and
some of it breaks playback. The Soloist backend learns to tell the app's actions from Music
Assistant's own, and the Spotify provider plays them on the Music Assistant queue: play now through
the queue's own play options, add to queue at the spot the app shows, next and previous as queue
commands. Music Assistant takes over the audio the engine already started for a pick, so nothing
restarts. No new Spotify endpoints, no polling.

## Decisions (2026-11-08)

1. **Play now** uses the queue's default enqueue option for the media type: album, playlist and
   artist replace the queue, a track plays now and keeps the rest.
2. **Add to queue** copies the engine's order: after the song Music Assistant already queued on
   the engine, else right after the current song. The Spotify app shows the same order.
3. **A context Music Assistant cannot look up** fills the queue from the engine's own `upcoming`
   list (decorations, no Spotify call), and falls back to the picked song alone when that list is
   empty.
4. **Always on**, no setting.
5. **Podcast episodes and audiobook chapters** are a later step (#409).

## Sources

- Test server, 2026-11-08. The dev add-on ran a throwaway branch that logged every Soloist
  WebSocket event and every command Music Assistant sent (decorations stripped). Soloist
  1.3.10.47 (build 20261105), a Sonos speaker, the Spotify app on a phone, the Music Assistant
  queue playing an album.
  - Round 1 (16:06-16:15 CET): two adds to the queue, an album from its 3rd track, a playlist, a
    song from search, a song radio, Liked Songs, a podcast episode, Next.
  - Round 2 (16:16-16:19): an add to the queue while Music Assistant's own next track was already
    queued on the engine, an album, an audiobook chapter.
- Probe traces from August 2026: the event order of Music Assistant's own commands, autoplay, and
  the full decorations of queue entries.

The event logs are not published; the findings below quote them.

## 1. What the engine reports

The engine never says who sent a command. Music Assistant tells its own actions from the app's by
what it asked for itself.

| Action in the app | Events (measured) |
|---|---|
| Play an album / playlist / radio / song / Liked Songs / episode | `context_changed` (new context) → placeholder `queue_changed` → `track_changed` of the first item 0.09-0.55 s later → `queue_changed` with `previous`/`upcoming` → `playback_state` |
| Add to queue | `queue_changed` only: a new `source:"queue"` entry, uid `qN` |
| Next, nothing queued on the engine | `playback_state` `paused` at 0:00 on the same track |
| Next, Music Assistant's next track queued | the engine moves onto that entry (August probe, `_consume_fed`) |
| Previous | not measured (the device was gone by then); August probe: < 3 s in goes to the previous item, paused at 0:00; > 3 s in restarts the track |

Findings the design relies on:

- **F1. Context URIs seen.** Album `spotify:album:6ZG5lRT77aJ3btmArcykra` (tapped track 3: the
  item is track 3 and `previous` lists tracks 1-2); playlist `spotify:playlist:37i9dQZEVXbKCF6dqVpDkS`
  (Top 50 - Netherlands); song from search `spotify:track:6N717o83wbmahFGUqkjQj7` (the context is
  the song itself, the same shape Music Assistant's own play has); song radio
  `spotify:playlist:37i9dQZF1E8Msndk0rmM1l` ("Radio Radio Ga Ga"); **Liked Songs
  `spotify:playlist:37i9dQZF1F5p3rmiWPIYgZ` named "Liked Songs", with an empty `upcoming`** (not
  `spotify:user:<id>:collection`); podcast `spotify:show:6C1XBOjNXCFLHVwPk6oXva` with a
  `spotify:episode:` item; audiobook `spotify:show:7b40X2fgBNosVOOtwLGSUt` with a chapter as
  `spotify:episode:`.
- **F2.** `context_changed` carries no name. The next `playback_state` carries it in the context
  decorations ("Parachutes", "Top 50 - Netherlands", "Liked Songs", ...).
- **F3. Music Assistant's own play has the same shape**, always with a `spotify:track:<uri it
  sent>` context. A fresh daemon also announces the account's restored item and context before
  Music Assistant's `play` (16:06:40, 16:13:05, 16:13:23), which the `_play_sent` gate already
  ignores.
- **F4. Queue entry uids come from one per-daemon counter**, shared by Music Assistant's
  `add_to_queue` and the app. Music Assistant's entry got `q0`, the app's add right after got
  `q1`. An add goes after the existing queue entries and before the context remainder:
  `upcoming = [q0 Music Assistant's next track, q1 the app's song, <context>]` (16:17:34).
- **F5. Queue entries survive** a context change by the app (`q0`, `q1` still at the head after
  the album play, 16:12:03) and Music Assistant's own `play()` (`q1` after `play(Maxwell)`,
  16:18:09). They vanish with the daemon: a new run starts with an empty queue.
- **F6.** An episode and an audiobook chapter started in the app both started at position 0 (one
  sample each, and the chapter had been partly heard before). Check again in #409.
- **F7.** Autoplay never changes the context (August: the album context stays while autoplay
  tracks play), so it can never look like a play from the app.

## 2. What the code does today

- `_handle_event` ignores `context_changed` (`soloist.py:1972-2037`; the model exists at
  `runtime.py:254`).
- **Pick, nothing of Music Assistant's queued on the engine.** `_observe_engine_item` takes the
  wander branch (`soloist.py:2097-2101`) and ends the item at the switch. The queue's next-item
  preparation then asks for Music Assistant's next item: `claim` fails, the run is not `busy`, and
  `move_to` sends `play(<Music Assistant's next>)`. Measured 0.11-0.74 s after the pick in six
  cases, and 3.8 s in one (the speaker still had the old track's tail to play). When Music
  Assistant's next item was already prepared, the run idles out instead after
  `_QUEUE_PLAYING_GRACE_S` ("Nothing more to play", 16:12:59 and 16:13:21) and kills the pick; a
  fresh daemon starts for Music Assistant's next item 5 s later.
- **Pick while Music Assistant's next track is queued and confirmed.** `_fail("Spotify is already
  streaming somewhere else…")` (`soloist.py:2090-2096`). Music Assistant queues its next track
  about 60 s before the end (`streams/audio.py:1891-1895`; measured: a seek to 190 s of 260 at
  16:16:42.9, queued at 16:16:50.2), so this covers the last minute of every song.
  music-assistant/server#7442 (open) holds that feed back until the last 8 s, which shrinks the
  window but does not close it. Code reading only: the round-2 pick came after the boundary.
- **Add to queue.** `_observe_queue` keeps the uid in `_queue_uids` (`soloist.py:2103-2124`), and
  `feed_next` refuses to feed while it is non-empty (`soloist.py:1013-1020`). At the next boundary
  the engine moves onto the app's entry and wanders, Music Assistant jumps to its own next track,
  and the entry is gone. Accepted in music-assistant/server#7384 as "one user track popped per
  boundary".
- **Next with nothing queued.** Paused at 0:00 mid-song: `_observe_rewound_pause` ignores it (not
  in the tail), `_observe_external_pause` starts the grace, and `_stop_queue_if_still_paused`
  stops the Music Assistant queue (`soloist.py:2190-2246`; measured 16:14:27.3 → 16:14:29.3
  "Spotify paused …, stopping queue").

## 3. Design

### 3.1 Where it lives

- **`_EngineRun`** is the only place that knows what Music Assistant asked the engine for, so it
  sorts the engine's reports into app actions. It does no Spotify lookups.
- **`SpotifyProvider`** gets one callback from the backend, `on_app_action(queue_id, action)`,
  turns Spotify URIs into Music Assistant media and drives the queue. It builds instance-scoped
  URIs (`f"{self.instance_id}://album/{id}"`); a bare `spotify:album:…` resolves to whichever
  Spotify instance comes first, which is wrong with more than one account.
- The callback runs as its own task (`mass.create_task`). Lookups and `play_media` take time, and
  the event handler must keep reading: command acks arrive through the same callback
  (`runtime.py:_send_command`), so a blocked handler times out every awaited command.
- **Core.** Nothing new for play now, next and previous. Placing an add-to-queue song needs
  `play_media(NEXT)` followed by `move_item`, or a small helper that inserts at an index if that
  reads cleaner. Decide in the build.

### 3.2 Sorting the engine's reports

A report is Music Assistant's own while `_moving`, before the first item started, or when it names
what Music Assistant expects (the current item, `_next`, the entry it expects next). Anything else:

- `context_changed` to anything but the `spotify:track:` context of Music Assistant's last own
  play → **play now**. Keep the context; the following `track_changed` names the start item. A
  `track_changed` to an unexpected URI mid-song without a context change is a play now within the
  current context (another track of the album the app started earlier).
- `queue_changed` listing a `source:"queue"` uid that Music Assistant did not add → **add to
  queue**, with its position among the queue entries.
- `paused` at 0:00 on the current item, mid-song, nothing queued → **next**. This replaces the
  queue stop for exactly this signature; a plain pause from the app keeps the
  music-assistant/server#7117 handling.
- `track_changed` to the URI of Music Assistant's previous queue item, within 3 s of the current
  one's start → **previous** (measure first).
- In the item's tail, the engine's own advance (context remainder, autoplay) stays a wander, as
  today. Autoplay never changes the context (F7).
- `device_changed` with `is_active=false` stays a takeover.

The sorting runs before the takeover check and the refusal check from
music-assistant/server#7436 in `_observe_engine_item` (`soloist.py:2084-2096`), so a pick is never
read as another device or as Spotify refusing Music Assistant's queued track (#408).

### 3.3 Keeping the pick's audio

On play now the run does what `_consume_fed` does for Music Assistant's own queued track: it ends
the current item at the switch and keeps the engine's new item as the unclaimed next one. Two
additions:

- **Hold.** While a pick waits to be taken up, a request for any other item waits in
  `_acquire_item` instead of moving the engine away. Waiting, not refusing: the queue's
  speculative next-item preparation could give up softly, but when a slow lookup outlasts the
  speaker's lead, the speaker asks for Music Assistant's next item for real, and a refusal there
  fails that item and the ones after it. The existing `_UNCLAIMED_ITEM_S` (10 s) bounds the hold.
  When the queue switches, the waiting request is cancelled with the old queue items. When Music
  Assistant never takes the pick up (a failed lookup), the waiting request proceeds and today's
  behaviour applies.
- **Take-up.** The provider calls `play_media(queue_id, <context>, option=None,
  start_item=<picked track>)`. The queue starts the picked track at 0, and its stream finds it
  through `claim` (same URI, no seek). The item's 10 s cushion holds what the engine rendered
  meanwhile; a full cushion suspends the sink, which pauses the engine. `refuse_held_item` only
  refuses for slots held by other queues (`stream_feeder.py:233-261`), so it does not block a play
  on the run's own queue.

The tracks after it are fed with `add_to_queue` as today. Fed entries play before the app's
context remainder, so the engine never reaches that remainder while Music Assistant feeds, and
every boundary stays a fed boundary.

`option=None` uses the queue's defaults: replace for album, playlist and artist, play for a track
(`player_queues/config.py:245-250`). Decision 1.

### 3.4 The engine's queue as an ordered list

`_fed` and `_queue_uids` become the ordered `source:"queue"` entries of the last report, each
marked as Music Assistant's or the app's, and marked once the provider has put it into the Music
Assistant queue:

- An app entry goes into the Music Assistant queue right after the Music Assistant item that
  precedes it in the engine's queue: the current item plus the number of entries ahead of it.
  That copies the engine's order, which is also what the Spotify app shows (decision 2).
- When the queue hints its next item and that item is the head of the engine's queue (a mirrored
  app entry), the run adopts the entry as its expected next one instead of sending
  `add_to_queue`. The boundary then runs through `_consume_fed` and `claim`, without a gap.
- An entry the user removed or moved in Music Assistant after mirroring is stepped over as today
  (one warm jump).

With music-assistant/server#7442, Music Assistant queues its next track only in the last 8 s, so
an add from the app nearly always arrives with nothing of Music Assistant's queued and goes right
after the current song. That PR also changes `feed_next` (it holds the hint until the feed is
due), which this section rewrites: build on it once it has landed.

### 3.5 Next and previous

`mass.player_queues.next(queue_id)` and `previous(queue_id)`. After a next with nothing queued the
engine sits paused at 0:00 on the old track, and Music Assistant's next track arrives through
`move_to` like any skip in Music Assistant.

## 4. Looking up what the app started

All lookups go through the provider's existing cached methods. No new endpoints, no polling.

| App pick | Context (measured) | Music Assistant lookup | Spotify calls, uncached | Cached |
|---|---|---|---|---|
| Song from search | `spotify:track:<id>` | `get_track` | 1 | 30 days |
| Album from track N | `spotify:album:<id>` | `get_album` + `get_album_tracks` | 1 + 1 per 50 tracks | 30 days / 1 year |
| Playlist | `spotify:playlist:<id>` | `get_playlist` + `get_playlist_tracks` | 1 + meta + 1 per 50 tracks | 30 days / 3 h by snapshot |
| Song radio | `spotify:playlist:37i9dQZF1E…` | as a playlist; Spotify-owned, so the global-token fallback (`provider.py:569-595`) | same | same |
| Liked Songs | `spotify:playlist:37i9dQZF1F…` named "Liked Songs" | unknown: probably not readable as a playlist; Music Assistant's own is `me/tracks` | ? | ? |
| Artist | not measured | `get_artist_toptracks`, or Music Assistant's artist play | 1-2 | 14 days |
| Podcast episode | `spotify:show:` + `spotify:episode:` | `get_podcast_episode` (+ resume position) | 1-2 | 24 h |
| Audiobook chapter | `spotify:show:` + `spotify:episode:` | audiobook + chapter offset | 1-3 | 30 days |
| Add to queue | entry `spotify:track:<id>` | `get_track`, or the entry's decorations | 0-1 | 30 days |

A large playlist resolves completely before playback starts (`media_resolver.py:358-410`): Liked
Songs with 2000 tracks, uncached, is 40 pages. That is the slowest case and the reason for the
hold in 3.3.

**Zero-call fallback.** Queue entries in `queue_changed` and `get_queue` carry name, artists (URI
and name), album (URI and name), duration and cover art (August traces). A `Track` built from them
needs no Spotify call, and streaming only needs the id. `get_queue` takes a `limit` (Music
Assistant asks for 10). For Liked Songs `upcoming` was empty when the pick arrived (Music Assistant
moved away 0.4 s later), so whether the engine fills it later is still open. Decision 3 falls back
to the picked song alone when it stays empty.

## 5. Fit with feed-ahead

- Music Assistant's queued track and the user's are told apart by uid (one counter, F4). Music
  Assistant's own add is matched to its uid by URI on the first report, as today; every other
  entry is the app's.
- The engine queue lives only as long as the daemon (F5). Mirrored entries are in the Music
  Assistant queue, so a daemon restart loses nothing.
- "One user track popped per boundary" goes away for mirrored entries. It stays for an entry
  changed in Music Assistant after mirroring.
- Music Assistant keeps feeding, so boundaries stay fed boundaries.

## 6. Edge cases

- **Endless Mix.** Add to queue is a NEXT track, which goes in as is even on a dynamic queue
  (`queue_loader.py:1014-1016`); the mix continues after it. Play now replaces the queue's sources
  like a play in Music Assistant does (`queue_loader.py:1103-1109`), so the mix ends, as it would
  when starting an album in Music Assistant.
- **Sonos group.** The run serves the group's queue (`streamdetails.queue_id`), and `play_media`
  on it restarts the group like a play in Music Assistant. Nothing special.
- **Account playing elsewhere.** Unchanged takeover through `_observe_device_active`; not
  followed.
- **Another Spotify account choosing the device on the LAN** (the daemon is visible there). Not
  measured. Follow only while the device stays active for the paired account; anything else stays
  a takeover.
- **Episodes (#409).** Music Assistant does not queue episodes on the engine, but an episode the
  app added is already in it; mirrored as NEXT, the engine reaches it by itself. For play now,
  Music Assistant's own resume position can differ from where the engine started (F6), so `claim`
  refuses and `move_to` seeks in place (0.2-0.5 s). Audiobook chapters map to the audiobook plus
  the chapter's offset as `start_position`.
- **Picking the song that already plays.** Same URI, ignored today; the engine restarts at 0 while
  Music Assistant's progress runs on. Treat it as a restart (seek 0) or leave it.
- **Transfer in** from the phone while Music Assistant plays: probably a play now with a non-zero
  position (not measured); `move_to` with a seek covers it.
- **Shuffle on in the app.** The engine starts on a random track; Music Assistant plays the
  context from that track with its own shuffle setting, and the engine's shuffle stays pinned off.

## 7. Tests

`tests/providers/spotify/test_soloist_backend.py` already drives `_handle_event` with built events
(`_queue_event`, `_track_event`, `_playback_event`), but has no `context_changed` event yet. Add
one and replay the recorded sequences: play now (album, track context, restored state at spawn),
add to queue with and without Music Assistant's entry queued, Next paused at 0:00, autoplay in the
tail.

## 8. Order of work

1. Build on `dev` once music-assistant/server#7442 has landed (or is dropped): it changes
   `feed_next`, which 3.4 rewrites. music-assistant/server#7436 is merged.
2. One server PR for #405, #406, #407 and #408: the sorting in the backend, the provider callback,
   the hold, the ordered engine queue, tests.
3. Measure before or during the build: previous, an artist pick, Liked Songs (does `upcoming`
   fill, can the playlist id be read), the latency of a large uncached playlist, a second account
   on the LAN.
4. #409 (episodes, audiobooks) after that.
5. #410 (docs) with the release that ships 2.
