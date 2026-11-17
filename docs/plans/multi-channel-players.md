# Multi-channel players: speakers that belong together are one player

Status: decided with the maintainer on 2026-11-17. Epic: #484. Owner: Marcel van der Veldt. This
is the technical companion of the epic on the project board, written so it can be fed to an agent
for implementation.

Claims marked "verified" were checked on server `dev` at `d4f339c4f`, models `main` at `bdb6a67`
and the current default branches of the clients. Paths are relative to `music_assistant/` in the
server unless they name another repository. The model of phase 2 is reviewed by the owner of the
players core before its technical design is fixed; it is marked as such below.

## Problem

Music Assistant cannot say "these speakers are one speaker".

- AirPlay registers every device as its own player. The two HomePods of a stereo pair are two
  players, and so are the HomePods an Apple TV uses as its speakers. An Apple TV in that setup
  stays silent (music-assistant/support#7021): it never passes a stream of ours on to its
  speakers, so the sender has to feed them.
- Sonos shows a stereo pair or a home theater set as one player, but nothing marks it as a set.
- Chromecast has a type of its own for a speaker pair, `PlayerType.STEREO_PAIR`, which no other
  provider uses.
- WiiM shows a pair made in its app as two players (music-assistant/support#6427).
- Two separate speakers become a stereo pair only through a sync group with a left or right
  setting per member. A group player cannot be a member of another group, and the members stay in
  every list (music-assistant/support#6126, music-assistant/support#6956).

## Concept

A **multi-channel player** is one player made of several speakers. It is played, grouped and
controlled as one whole in the interface, in Home Assistant and in groups. Its speakers are not
players of their own and there is no general option to show them next to it. Where a kind of set
needs a choice, that is a player option of that set.

A set comes to exist in one of two ways:

- **Delivered.** The provider knows the set and registers one protocol for it. Sonos and
  Chromecast do this today; AirPlay joins them. The core only sees the type and the layout.
- **Composed.** The user builds a set from existing players and gives each a channel.

## Decisions (2026-11-17)

1. `PlayerType.MULTI_CHANNEL` (`multi_channel`) replaces `STEREO_PAIR`. The layout (stereo, 2.1,
   5.1) is a property of the player, not a type. `STEREO_PAIR` stays in the enum for old data and
   is no longer sent.
2. AirPlay delivers its sets itself, like Sonos and Chromecast: one protocol per set, the speakers
   managed inside the provider. No sync group, no core flag for it.
3. Composing lives in a new builtin provider next to the sync group and the universal group, with
   a small piece of generic support in the players controller.
4. A delivered set continues on one existing player: the Apple TV for a home theater set, the
   first speaker for a pair. Its id, Home Assistant entity, queue and settings stay. The players
   of the other speakers are removed, as Chromecast does today, and return as new players when
   the set is taken apart.
5. Phase 1 is the general support and the delivered sets. Phase 2a composes a stereo pair, 2b
   adds a subwoofer (2.1, 1.1), 2c is multi-channel audio in the core and in composed players,
   possibly as its own epic. Nothing chosen for phase 1, 2a or 2b may rule out 2c.
6. The speakers of an AirPlay set each get the full stereo stream: a HomePod of a pair plays its
   own channel by itself.
7. music-assistant/server#7174 is not merged. Its record parsing and its "go on without a failed
   member" change are welcome as PRs of their own.

## Phase 1: general support (#485)

**models** (one additive release)
- `PlayerType.MULTI_CHANNEL = "multi_channel"`.
- `Player.channel_layout`, optional. Values follow ffmpeg's channel layout names where one exists
  (`stereo`, `2.1`, `3.0`, `3.1`, `5.0`, `5.1`, `7.1`), plus `1.1` for one speaker with a
  subwoofer; an unknown value falls back the way the other enums do. Empty for a player that is
  no set, or whose layout its provider cannot tell.

**server**
- `PlayerProtocol` reports the layout; the state derivation publishes it.
- The type takes the place of `STEREO_PAIR` in the six lists that name it (verified):
  `helpers/config_entries.py:31` (playback targets), `controllers/players/constants.py:71`
  (ungroup on power off), `helpers/player.py:75` (icon), `controllers/players/lifecycle.py:372`
  (no MAC lookup), `controllers/config/players.py:1261` (audio config entries),
  `providers/sendspin/player.py:191` (line-in targets).
- A stored or reported `stereo_pair` is read as `multi_channel`, where `Player.reported_type`
  (`models/player.py:966`) already turns `PROTOCOL` into `PLAYER`. The stored type of a player is
  rewritten when the reported one differs (`controllers/players/lifecycle.py:935`), so no
  migration step is needed.
- Two group volume branches test `type != PLAYER` and so leave the own level of a set that leads
  a session out of the group volume (`controllers/players/volume.py:419`,
  `models/player_state.py:1301`). They test for a group player instead.
- API schema version: the shared model changes shape, so it is raised; take a number that no
  open branch holds.
- The README and SCENARIOS of the players controller describe the type.

**Clients** (verified: none breaks on a type it does not know)
- Home Assistant pins `music-assistant-client==1.8.0` with models 1.1.236; `PlayerType._missing_`
  gives `UNKNOWN`. The integration only branches on `GROUP` (volume) and exposes the type as the
  attribute `mass_player_type`, which reads `unknown` until the client is bumped. Player options
  already become switch, select and number entities.
- The frontend ships with the server: `player_type.multi_channel` label, the filter option that
  `stereo_pair` has in `PlayerFilters.vue`, a layout badge, and the grouped icon, which
  `PlayerCard.vue`, `PlayerFullscreen.vue` and `PlayerTrackDetails.vue` only give to `PLAYER`.
- The mobile app maps an unknown type to a plain player (`PlayerFactory.kt`), the tvOS app decodes
  it to `.unknown` and lists it. Both get the value for completeness; the mobile app as a draft
  PR for its maintainers.

## Phase 1: delivered sets

### AirPlay: an Apple TV with HomePods (#486)

Measured on an Apple TV 4K (tvOS 27.2) with a HomePod mini (OS 27.2) as its speaker:
- Through the Apple TV the HomePod stays silent whatever is sent (realtime, buffered, relay
  keys). A stream straight to the HomePod plays, also in one session with the Apple TV.
- An Apple TV session opened before the HomePod session is reset by the Apple TV when the HomePod
  session comes up; HomePod first and the Apple TV afterwards, both stay.
- The set shows in `_airplay._tcp`: the Apple TV carries `psgid`, `psgsz`, `psgtp=1`, `pgmid=0`
  and `gpn` (the room name); its speaker the same `psgid`, `pgid` equal to it, and `pgmid` above
  zero. An Apple TV without such speakers carries none of these keys.

Design:
- The provider reads these keys. The protocol of the Apple TV becomes the set: type, layout, the
  name from `gpn`. The HomePods are fed directly, HomePods first, with one volume for them.
- The protocols of the HomePods are removed with `players.remove_protocol` once the set is seen
  for certain (the Apple TV and the speaker name the same `psgid`), never because a record is
  missing. A speaker that advertises itself without the set again is registered again. Their
  pairing data and per-speaker values live with the set while it exists.
- The Apple TV takes part as a member without sound. Player option on the set: "Sound only
  through the connected speakers". Its default, and what the Apple TV does while it is off, are
  decided with the maintainer after the open measurements below.
- The Sendspin bridge rides on the protocol of the set.
- From music-assistant/server#7174: the record parsing (`apple_groups.py` with its tests) and
  the change that lets a session go on without a member that fails to connect.

To measure before building: what the TV shows, the Apple TV in standby, pause, seek and next, a
HomePod pair on an Apple TV (`psgsz=3`).

### AirPlay: a HomePod stereo pair (#491)

- Known from the tests of music-assistant/server#7174 (HomePod 2018, AudioOS 17.2): both speakers
  carry the same `tsid`, and `gid` is `<tsid>+<0 or 1>+<device id>`. To confirm on a current OS.
- The same construction keyed on `tsid`: the protocol of the first speaker becomes the pair, the
  other speaker's protocol is removed while the pair exists. Both get the full stereo stream.
- The HomePods of a pair that belongs to an Apple TV carry both `psgid` and `tsid`: the Apple TV
  set wins.

### Sonos and Sonos S1 (#487)

- Verified: the provider registers only the primary of a set (`providers/sonos/provider.py:233`
  skips a device without the playback capability; `providers/sonos_s1/provider.py:171` skips an
  invisible one). aiosonos carries the channel map of the zone's speakers and whether a sub and
  surrounds are present, which the provider only uses for its sound options.
- The primary reports `MULTI_CHANNEL` and the layout when its zone has more than one speaker: a
  pair is `stereo`, a soundbar or amplifier with a sub and surrounds the matching layout. The
  type follows changes made in the Sonos app.
- The path that adds a speaker by IP address (`providers/sonos/provider.py:170`) gets the same
  check as discovery, so a satellite is not registered.

### Chromecast (#488)

- Verified: a multichannel group is one protocol of type `STEREO_PAIR` with its members cleared;
  its children are never registered, or removed when found later
  (`providers/chromecast/provider.py:238` and `:299`).
- It reports `MULTI_CHANNEL`, with the layout `stereo` for a pair. What the devices report for
  other multichannel groups is checked; the code path is the same for all of them.

### WiiM (#489)

- Verified: the provider has no handling; a native pair shows as a leader and a follower.
- The provider recognises a native pair or multichannel set from what the device reports and
  shows it as one player on its leader; the followers are not registered while they belong to
  it. The device divides the channels itself.

### Raumfeld and other brands (#490)

- Raumfeld already shows a room with several speakers as one player; it reports type and layout.
- One short check per remaining provider, Bluesound first (its library parses fixed groups
  already), then HEOS, MusicCast, Bose SoundTouch, Samsung, Yandex Station and Alexa. A provider
  comes in when it can tell from data it already holds, without extra requests, and someone can
  confirm it on a real set. Otherwise it waits for a request.

## Phase 2: composed sets

The model below is the direction. Its technical design is fixed after the review by the owner of
the players core, and this section is updated then.

Verified starting point: a `Player` holds at most one protocol per domain and one protocol that
stands for the device (`controllers/players/protocol_linking.py:203` and `:217`, the identity
veto in `controllers/players/matching.py`, the merge check in `controllers/players/merging.py`).
Output selection, `play_media`, the state derivation and the grouping plan all work with one
protocol per domain. That rule stays.

**Model**
- A builtin provider creates the composed player through the existing group player flow
  (`ProviderFeature.CREATE_GROUP_PLAYER`, `players/create_group_player`, the frontend's
  `AddPlayerGroupDialog.vue`) and registers one composite protocol for it, of type
  `MULTI_CHANNEL`. Its member table names per speaker a channel.
- The composite stands in the single place per domain and drives its speakers: it forms the
  protocol's own sync set of them over the one output they all share, picked the way a group
  picks it today, and sends volume, mute and power to each. To the rest of the core the composed
  player is a plain player, so it joins sync groups and universal groups and can lead a session.
- Generic support in the players controller:
  - an explicit *absorbed* state for the players of the members: not live, not listed, not a
    candidate for matching, with their records, settings and queues kept, and back when the set
    is removed. A disabled player is not this: its protocols attach nowhere and it stays listed;
  - the protocols of the members belong to the composed player while it exists, with the player
    they came from remembered, so removing the set is an exact undo;
  - the executor of a group change expands a composed member into its speakers when it calls the
    protocol of another leader, and never treats the composite's own speakers as members to move;
  - `controllers/streams/audio.py::_get_output_channels` takes the channel from the member table.
- Members have to be able to sync with each other. Browser and app players are refused.
- How tight the speakers play together is that of the protocol they share.

**Stored data.** The records of a composed player are new. The link of a member protocol to its
player moves and is restored on removal: a change to existing records made when the user
composes, not at an upgrade. The session that builds it puts the migration question to the
maintainer before it is done.

**2a: stereo pair (#492).** Two members, left and right. No new settings: balance is the existing
DSP filter, delay the existing per-protocol setting, shown under each member.

**2b: subwoofer (#493).** A subwoofer member gets a low-passed mix of both channels. Two settings:
crossover frequency and subwoofer level. A single speaker with a subwoofer is the 1.1 case. This
stays inside the stereo pipeline.

**2c: multi-channel audio (#494).** Sources with more than two channels, an upmix of stereo, and
a pipeline and DSP that carry the channels. Verified today: the PCM, flow, realtime and
announcement formats cap at two channels (`controllers/streams/audio.py`, `helpers/audio.py`),
and the DSP knows the channels all, front left and front right. Not designed here.

**Kept open for 2c**
- The layout uses ffmpeg's layout names.
- The channel of a member is a channel of that layout: the existing `AudioChannel` enum of the
  models, extended with `LFE` in 2b and the surround channels later. Never a filter string.
- Every speaker of a set keeps its own output pipeline, as the members of a sync session have
  today, so 2c changes what feeds a speaker and not how a set is built.
- The subwoofer is fed by bass management, which later also takes a real LFE channel.
- No new API or stored value says "two".

## Verification

- Every guaranteed behaviour gets a scenario with a test id in
  `controllers/players/SCENARIOS.md`: the type and layout, the group volume of a set that leads,
  a set as a member of a sync group and of a universal group; for phase 2 also a restart in any
  order, a member that arrives late or is removed, and a removal that restores everything.
- End-to-end lab with virtual players: a composed pair of two Sendspin players, alone and in a
  group, left and right checked in the captured audio.
- Real devices for AirPlay: an Apple TV with a HomePod as its speaker (start, pause, seek, next,
  the option, standby), and a HomePod pair.
- Sonos, Chromecast, WiiM and the other brands: unit tests on recorded device data, and a user
  who confirms it on a real set where the test park has none.
- Clients: Home Assistant shows one entity per set with the right type and the option as a
  switch; a client that does not know the type keeps working.
