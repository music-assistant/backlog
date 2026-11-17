# Multi-channel players: speakers that belong together are one player

Status: decided with the maintainer on 2026-11-17. Epic: #484. Owner: Marcel van der Veldt. This
is the technical companion of the epic on the project board, written so it can be fed to an agent
for implementation.

Claims marked "verified" were checked on server `dev` at `d4f339c4f`, models `main` at `bdb6a67`
and the current default branches of the clients. Paths are relative to `music_assistant/` in the
server unless they name another repository. The owner of the players core reviewed the model
against the code on 2026-11-17; phase 1 and phase 2 below carry what came out of that. The
sections on the delivered sets also carry what was built and measured that day (server `dev` at
`fb8e5bcbe`).

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
   of the other speakers are removed, with their settings, and return as new players when the
   set is taken apart.
5. Phase 1 is the general support and the delivered sets. Phase 2a composes a stereo pair, 2b
   adds a subwoofer (2.1, 1.1), 2c is multi-channel audio in the core and in composed players,
   possibly as its own epic. Nothing chosen for phase 1, 2a or 2b may rule out 2c.
6. The speakers of an AirPlay set each get the full stereo stream: a HomePod of a pair plays its
   own channel by itself.
7. music-assistant/server#7174 is not merged. Its record parsing and its "go on without a failed
   member" change are welcome as PRs of their own.

## Phase 1: general support (#485)

**models** (released in 1.1.243)
- `PlayerType.MULTI_CHANNEL = "multi_channel"`.
- `Player.channel_layout`, optional, with the enum `ChannelLayout`: the number of full-range
  speakers and of subwoofers of the set in home theater notation (`1.1`, `2.1`, `3.0`, `3.1`,
  `4.0`, `4.1`, `5.0`, `5.1`, `7.1`), and `stereo` for a set of two speakers. An unknown value
  falls back the way the other enums do. Empty for a player that is no set, or whose layout its
  provider cannot tell.
- A new field on the models `Player` needs its property on the server's `Player` in the same
  step (`tests/models/test_player_final_values.py` checks every field), so the server change
  carries the models pin.

**server**
- `PlayerProtocol` reports the layout. The player publishes the layout of its own protocol, as it
  does for the type. It is part of the protocol inputs, the state inputs and the state
  fingerprint, or a change of the layout alone is never published. While the own protocol is
  away the layout is empty: it is not stored.
- The type takes the place of `STEREO_PAIR` in four lists: `helpers/config_entries.py`
  (playback targets), `controllers/players/constants.py` (ungroup on power off),
  `helpers/player.py` (icon) and `providers/sendspin/player.py` (line-in targets).
- Not by type: the MAC lookup at attach (`controllers/players/lifecycle.py`) is skipped for a
  group player only. A set that rides on the protocol of a real device (an Apple TV, a Sonos
  primary, a WiiM leader) keeps its MAC and with it the matching of that device's other
  protocols. A Cast group reports no address, so nothing is looked up for it. The type entry in
  `controllers/config/players.py` had no effect and is gone.
- Old values: `Player.reported_type` reads a `stereo_pair` as `multi_channel`, next to the rule
  that turns `PROTOCOL` into `PLAYER`. The config listing of a player that is not live reads the
  stored record, so a one-time migration step rewrites a stored `stereo_pair`.
- A group player has no level of its own; every other leader counts with its own level in the
  group volume. The two branches that decide it (`controllers/players/volume.py`,
  `models/player_state.py`) test for the group player.
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

## Phase 1: the other speakers of a set

A speaker that goes up in a set may have been a player before: it was set up before the upgrade,
or it was bonded while the server runs. Its provider no longer registers it, but its player stays.

Measured on Sonos (a stereo pair taken apart and bonded again, a server on `dev` and one on a
nightly build):
- The player of the bonded speaker stays listed and reads available: its connection stays up,
  although the speaker cannot play on its own.
- `players.remove_protocol` removes a player only when no other protocol record of it is left,
  and nearly every speaker also has an AirPlay record.
- A removal by the user lets every provider of that player look again, which replays the
  announcements the server still holds: the AirPlay provider set the speaker up again within 20
  seconds, from a record the speaker no longer sent. That lasted until the record expired,
  about an hour and a half later.

So the players controller gets one rule for all brands: the provider of a set reports its other
speakers, and the core removes their players.

`await mass.players.remove_absorbed_players(protocol_id, absorbed_protocol_ids)`

- `protocol_id` is the protocol that delivers the set. `absorbed_protocol_ids` holds the
  provider's own protocol ids of the other speakers of that set. The provider calls it when the
  protocol is set up and whenever the set changes. Calling it again is free: a speaker whose
  player is already gone is skipped.
- For each id that has a stored record with a player, that player is removed for good, as when
  the user removes it: its protocols, settings, DSP, saved queue and group memberships. Its
  providers are not asked to look again, which is what brought the speaker back in the
  measurement above.
- Skipped: an id without a record, an id of another provider than the set's protocol, the set's
  own player, a group, and a player that is being removed already.
- Nothing is stored and nothing is hidden. A speaker that leaves its set is found by normal
  discovery and is a new player.
- It cannot be undone, so the provider must be sure: the set itself names the speaker as one of
  its speakers at that moment, never a speaker that is silent, a timeout or an incomplete list.
  It names only its own protocol ids, and it registers no protocol of its own for a speaker
  while its set names it.
- Accepted: a protocol of another provider that still finds the speaker gets a new, visible
  player, which the user removes. Bonding speakers while the server runs is a one-time setup
  step, so no list of devices to keep out and no waiting time are built for it.

What follows from it:
- A speaker that plays its own queue at that moment is stopped, and one that sits in a group
  leaves it, as on any removal. No power off goes out.
- What a plugin keeps per player goes as for any removal.
- Home Assistant drops the device of the speaker.

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
- A speaker is never registered while its set is seen: the provider filters it where it handles
  discovery. AirPlay sets up every advertised device it does not hold, so without that filter
  every flap of a record costs a player.
- A speaker that was a player before is reported with `remove_absorbed_players`, on a positive
  identification of the set (the Apple TV and the speaker name the same `psgid`). Pairing and
  other setup data the set still needs are copied to the set first, because the speaker's
  record goes with its player.
- A speaker that advertises without the set again is registered by normal discovery, as a new
  player.
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
- The same construction keyed on `tsid`: the protocol of the first speaker becomes the pair, and
  reports the other speaker with `remove_absorbed_players`. Both get the full stereo stream.
- The HomePods of a pair that belongs to an Apple TV carry both `psgid` and `tsid`: the Apple TV
  set wins.

### Sonos (#487)

- On `dev` (music-assistant/server#7863): the primary reports `MULTI_CHANNEL` and the layout
  while its zone has more than one speaker, from what the speaker already reports:
  `has_subwoofer`, `has_surrounds` and `is_stereo_pair` of aiosonos, the home theater capability
  of the device and the Amp model.
- Layout: the front counts 1 for a single speaker, 2 for a stereo pair or an Amp, 3 for a
  soundbar; surrounds add 2; a sub adds ".1". A pair is `stereo`, a soundbar with a sub and
  surrounds `5.1`, an Amp with surrounds `4.0`. A speaker, soundbar or Amp alone is no set.
- The provider registers only the primary: a device without the playback capability is skipped
  at discovery and on the path that adds a speaker by IP address.
- Confirmed on real speakers: two stereo pairs (`stereo`), a Beam with two surrounds (`5.0`),
  and a pair taken apart and bonded again in a running server (a plain player within 7 seconds,
  a set again within 10).
- What the speakers report: a pair has two zone members with the channel maps `LF,LF` and
  `RF,RF`, a soundbar with surrounds `LF,RF`, `LR` and `RR`, a speaker alone one member with an
  empty map. The ids of the zone members are the ids the provider uses as protocol ids.
- To build: the primary reports the other speakers of its zone with `remove_absorbed_players`,
  at setup and when its zone changes, from `bonded_player_ids` of aiosonos (0.4.0).
- Sonos S1 is not built. SoCo's public `channel` is never cleared when a pair is taken apart, so
  S1 cannot follow a pair. It stays as it is until someone asks.

### Chromecast (#488)

- On `dev` (music-assistant/server#7862): a multichannel group is one protocol that reports
  `MULTI_CHANNEL` with the layout `stereo`, with its members cleared; its children are never
  registered, or removed when found later (`providers/chromecast/provider.py`).
- Checked: a device only says whether a group is a multichannel group (`eureka_info`, read in
  `providers/chromecast/helpers.py`), not which speakers it has or which channel each plays, and
  only left and right pairs are known to exist. The layout stays `stereo`.
- Later: `remove_absorbed_players` in place of its own removal of a child that is found later.

### WiiM (#489)

- Reported with the answers of the speakers in music-assistant/support discussion 6427: the
  leader of a pair made in the WiiM Home app lists its other speaker with `group_channel` 2 and
  reports `group_type` 2; the other speaker reports itself as a slave. Both show as players
  today.
- Built on a branch, not merged: a slave whose `group_channel` is above zero belongs to the set
  of its leader. One such slave gives `stereo`, anything else `unknown`. The leader is the
  multi-channel player; the slave of its set is kept out of discovery, out of the group members
  and out of the players to group with. A slave only knows the id of its leader, so a follower
  that is found before its leader is looked at again once the leader was read.
- Open, asked from the reporter: what an ordinary group reports in `group_channel` on current
  firmware, and what a pair looks like that joins the group of another speaker. `group_type`
  would be the cleaner sign, but the library does not pass it on.
- To build: the leader reports the slaves of its set with `remove_absorbed_players`.

### Raumfeld and other brands (#490)

- Raumfeld already shows a room with several speakers as one player. Its type and layout are
  asked from the maintainer of that provider, who has the speakers.
- A provider comes in when it can tell from data it already holds, without extra requests, and
  someone can confirm it on a real set. Otherwise it waits for a request. Checked on 2026-11-17,
  none comes in yet:

| Brand | What Music Assistant shows for a set | What the provider can tell |
|---|---|---|
| Bluesound | one player per speaker | which speakers form a fixed group and its leader (`zone`, `zone_leader`, `zone_follower` of pyblu, read on every poll), not their channels. First in line when someone reports such a set |
| HEOS | one player per entry of the device list | nothing: the library has no field for a pair |
| MusicCast | one player per speaker | only that a device can pair, not that it is paired |
| Bose SoundTouch | one player per speaker | nothing without an extra request |
| Samsung | one player per speaker | the device sends its channel, the library drops it |
| Yandex Station | one player per speaker | nothing |
| Alexa | one player per device entry | the members of an entry, not whether it is a pair or a multiroom group |

## Phase 2: composed sets

Verified starting point: a `Player` holds at most one protocol per domain and one protocol that
stands for the device (`controllers/players/protocol_linking.py`, the identity veto in
`controllers/players/matching.py`, the merge check in `controllers/players/merging.py`). Output
selection, `play_media`, the state derivation and the grouping plan all work with one protocol
per domain. That rule stays.

**Model** (the cut of the players core review)
- A builtin provider creates the composed player through the existing group player flow
  (`ProviderFeature.CREATE_GROUP_PLAYER`, `players/create_group_player`, the frontend's
  `AddPlayerGroupDialog.vue`) and registers one protocol for it, of type `MULTI_CHANNEL`.
- Membership is a relation on the record of that protocol: the player ids of the members, under
  the key that group players use, which a removal and a merge of a player already keep right.
  The link of a member protocol to its own player stays where it is. Removing the set touches
  only the set's own protocol, so the members are back as they were.
- The players of the members are *absorbed*. That is the disabled state with three differences:
  their protocols stay wanted, so the providers keep them connected; they are hidden from the
  config listing; enable, merge and unlink refuse. One stored marker on the member's own record
  says so. They stay candidates for matching, so the other protocols of a member device keep
  landing on its own player and are there again when the set is removed. This is not the rule
  of phase 1: a delivered set removes the players of its speakers, a composed set keeps the
  players of its members and gives them back.
- One gate at attach, beside the check whether a protocol is wanted and before the player of a
  protocol is resolved: a protocol of an absorbed player is lent to the set when the relation
  uses it, and is stored as a link only when it does not.
- Grouping: the set's protocol declares the domain it groups in, read where the outputs of a
  player, the grouping snapshot and the "can group with" list are built. The snapshot folds the
  ids of the speakers into the set, the plan rules work with the set, and the executor expands
  it into its speakers at the call to the device. The set never reports its speakers as group
  members. Only the controller's `apply_group_change` calls a protocol's grouping methods; the
  set's provider goes through the controller, as the sync group does.
- A foreign protocol is kept off a group player by its type today; for a composed player that
  guard keys on the virtual role of its own protocol, as the merge check already does.
- `controllers/streams/audio.py::_get_output_channels` takes the channel from the member table.
- Members have to be able to sync with each other. Browser and app players are refused.
- How tight the speakers play together is that of the protocol they share.

**First.** The removal of a group member whose shared output protocol stays registered and reads
unavailable does nothing today (`controllers/players/grouping_plan.py`); a member of a set takes
that path, so it is fixed before phase 2.

**Stored data.** New: the record of the composed player and of its protocol with the member
relation, and the marker on each member's own record. No existing link moves. The session that
builds it puts the migration question to the maintainer before it is done.

**To decide with the maintainer at the start of phase 2** (the code does not settle these):
- the queues of the members while they are absorbed;
- whether the id of a member resolves to the set, for clients and automations that still name it;
- user filters and allow-lists that name a member;
- per-member settings and DSP: a player that is not live has no config entries, the DSP of a
  speaker's stream resolves to the player its protocol is attached to, and the per-protocol
  output channel setting competes with the member table;
- a member behind its own power control;
- whether the layout is stored, so it shows while the device is away.

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
- The layout is an open list of speaker counts; a new one needs no change of the model.
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
