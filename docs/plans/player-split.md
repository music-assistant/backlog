# Player split: one player per device, providers attach protocols

Status: decided with the maintainer on 2026-11-09. Epic: #412. Additions from the review of
2026-11-09 are marked *(review)*; where one changes a decision, the decided text stays and the
change follows it as *(review: proposed change)*.

Naming *(review)*: `Player` is the player (what `mass.players.get_player()` returns).
`PlayerProtocol` is the provider object: one instance per device per protocol domain, and the
vendor's own control plane is a protocol like any other. Native is a role of a protocol, not a
separate kind. The API already models the native output as one of the protocols
(`OutputProtocol.is_native`), so the UI vocabulary ("Output protocols"), the config keys and the
README stay. The words "core player" and "endpoint" are not used.

## Problem

A `Player` today is two things in one object: the player that users, Home Assistant and the queue
see (id, config, queue, controls, groups, output choice, derived state), and the provider's live
handle on a device. A player therefore only exists while a provider has it registered.

- A switched-off device disappears whenever its provider drops it: Squeezelite, Sendspin and
  AirPlay on disconnect, MSX after an idle timeout, every provider on a reload, and every native
  provider after a server restart. Power on through a smart plug (music-assistant/server#7435)
  cannot reach it, and Home Assistant cannot call it either (HA skips unavailable entities in
  service calls).
- State the player owns lives on the provider's object and dies with it: linked protocols, active
  output protocol, sleep timer, fake power/volume/mute. `register_or_update` loses the protocol
  links on a swap.
- Devices without a native provider need a stand-in identity, which is the universal player
  provider (710 lines) plus about 1,200 lines of linking, merge and takeover in
  `protocol_linking.py`. A native player taking over a universal player changes the id: HA gets a
  new entity, the saved queue is purged, Spotify Connect, AirPlay Receiver, Plex Connect, party and
  Yandex silently lose the device.
- Four providers unregister on device loss (Squeezelite, Sendspin, AirPlay, MSX), the rest stay
  registered and flag unavailable, so the same situation behaves differently per provider.
- *(review)* The support issues of the last six months (69 relevant) group into: group membership
  and leader lifecycle (19), wrong or duplicate identity from identifier matching (10), device
  offline or reconnect not handled (9), volume or controls routed to the wrong protocol (8), id and
  HA entity churn (7), power control vs availability (5), stale or unapplied configs (5). The model
  needs a rule for each, not only for the id churn.

## Target model

**Player** (owned by the players controller, what `mass.players.get_player()` returns). Identity,
config, queue binding, power/volume/mute control resolution, output protocol selection, group
membership, sleep timer, transient flags, and the state derivation (today's `@final` half of
`models/player.py`, about 90 members). Persisted device data: name, device info, identifiers
(universal players already persist these). Created from config at startup, or by the first
protocol of a new device. Removed only when the user removes it or disables it.

**PlayerProtocol** (provider object, today's provider half: `_attr_*` state, commands, config
entries, setup flows). Roles, reported by the protocol itself because one class can play several
roles per device (Chromecast, Squeezelite, Sendspin, AirPlay):
- native: the vendor's control plane, usually with its own audio path;
- control: control and grouping without an audio path (WiiM/LinkPlay over DLNA today);
- audio: an audio path (AirPlay, Cast, DLNA, Sendspin, Squeezelite bridges);
- virtual: no device (sync group, universal group, auracast).

Every device is one player, with no exceptions: native devices, protocol-only devices (today's
universal players), Cast groups and stereo pairs, sync groups and universal groups (virtual
protocols). A player has at most one native protocol and at most one protocol per domain; none is
required.

A protocol attaches to exactly one player. One persisted edge records it: the protocol's
`protocol_parent_id` (already the canonical side today; the key name stays to avoid a data move),
stored on every protocol whose id differs from its player's id. Protocol configs stay where they
are (`players/{protocol_id}`). At startup each player builds its protocol registry from the
protocols that point to it, so a protocol belongs to one player by construction. The parent-side
`linked_protocol_ids` list is no longer read or written.

**Player id.** Owned by the player, stable for its whole life, never derived again and never
switched by a protocol. A one-time migration freezes the current visible id of every existing
player (RINCON_…, Cast uuid, `up…`) as its player id, so HA entities, user filters, groups, DSP,
queues and plugin bindings stay as they are. A player created after the change gets an
MA-minted opaque id. Moving existing players to MA-minted ids, with a clean HA migration, is a
later step.

**Legacy id.** *(review)* Every player gets a `legacy_id`: the id it had before the split (for a
frozen id the id itself; after #418 the pre-move id), plus any id absorbed by a merge. The API
resolves a legacy id wherever a player or queue id is taken. `PlayerState` exposes it. The HA
migration of #418 maps `unique_id` and the `(DOMAIN, player_id)` device identifier on it.

**Three layers.** *(review)* Not merged into one. `PlayerConfig` is what is persisted and edited.
`Player` is the live object that owns identity, config, the protocol registry, the queue binding
and transient flags, and that decides. `PlayerState` is the published snapshot (on the wire
already named `Player` in the models package) and stays a plain serialisable dataclass. The state
derivation (today's `__final_*` methods) becomes a pure function of the protocols' reported state,
the config and the queue, called from `update_state`, which keeps the fingerprint diff and the
single `PLAYER_UPDATED` per change. `Player` becomes thin, and the derivation is testable without
provider objects.

**Ownership.** *(review)* A player is not owned by a provider instance. `PlayerState.provider`
names the primary protocol's provider instance (native while attached, else the highest-priority
audio protocol), is recomputed on attach and detach, and always resolves to a loaded provider.
The players controller handles enable, disable and remove of a player; today
`on_player_config_change` routes them through `mass.get_provider(config.provider)` and silently
does nothing when that lookup fails. The player owns its own config entries (name, controls,
output selection, DSP, queue); protocol entries are injected per protocol as today. Removing a
provider deletes that provider's protocol configs; a player left without any protocol config is
removed with them (music-assistant/support#5610: WiiM provider removed while AirPlay and Sendspin protocols
remained, the player vanished from HA).

**Config shape.** *(review)* With frozen ids, `players/{id}` is both the player record and the
native protocol's record, while a player minted after the change has a separate record: two shapes
for the lifetime of the code (config entry resolution, enable cascade, DSP and queue keys,
migrations, tests). Proposal: `players/{player_id}` becomes the player record only (name, enabled,
identifiers, device info, legacy id, last seen, controls, preferred output, memberships); protocol
instances move to `player_protocols/{protocol_id}` (provider values, setup data, role, `player_id`
as the edge, replacing `protocol_parent_id`). For a frozen id the two records share the id string
but not the key. The migration splits the current record by key (provider-specific values move,
user-level keys stay) and is idempotent; rollback is not required. Alternative: a new tree for
players, with protocol configs left where they are. Decide explicitly, also if the answer is "two
shapes".

**State of a player with several protocols.** *(review)*
- `available` = any attached protocol `available_for_playback`; `needs_setup` is aggregated the
  same way. Today only `ProtocolBackedPlayer` does this; a native parent goes unavailable when its
  native protocol drops although AirPlay and Cast still work (music-assistant/support#6951).
- Control target per feature: the active output protocol, else native, else any available protocol
  (music-assistant/support#6468: volume went to the idle snapclient while Sendspin played; music-assistant/support#6907:
  `active_output_protocol` never set when a member joins through a native AirPlay group).
- Name: custom, else native protocol, else audio protocols by priority, else last known (the rule
  of music-assistant/server#7010 moved into the player).
- Device info and identifiers merged from every protocol, ARP MAC included, and persisted.
- Power is tri-state: `powered=None` is unknown, never off (music-assistant/support#6823).
- The player owns volume and mute; a protocol receives them on activation. Protocol caches that
  re-assert their own level at stream start or scale differently per protocol are music-assistant/support#5929,
  music-assistant/support#5529, music-assistant/support#6110 and music-assistant/support#6664.

**Protocol re-identification.** *(review)* Protocol ids change too, and the persisted edge hangs on
them: ESPHome Sendspin MAC → key id (music-assistant/server#6928), Raumfeld room UDN ↔ serial
(music-assistant/server#7350), an ESP32 re-flashed from Squeezelite to Sendspin with the same MAC
(music-assistant/support#5636), two id schemes for web players (music-assistant/support#6773). Rule: a new protocol whose
identifiers match a player that has no attached protocol of its domain, or an offline one from the
same provider, attaches to that player and inherits the offline protocol's config (generalise
`carry_over_player_config`); the stale protocol config is dropped. The per-provider takeover code
goes with it.

**First sight and merge.** *(review)* An audio protocol that matches no player waits a short
collect window (today's 15 s) before a player is minted, so the device's other protocols land on
the same player. Identifiers are often incomplete at discovery (ARP pending, a relative
`presentationURL` music-assistant/support#6481, Cast identification failure music-assistant/support#6134, a TV advertising
`localhost.local` music-assistant/support#6998); without the window that is two players and a merge, and a merge
deletes an id (music-assistant/support#5549 flips entity ids on half the restarts). Auto-merge only while the
younger player is unused: created this session, no custom config, never played, in no group.
Otherwise both stay and the UI offers "merge into…". The absorbed id becomes a legacy id.

**Where one protocol per domain is not enough.** *(review)* A native protocol can claim further
instances of one domain, attached hidden (a Raumfeld room of three DLNA renderers, music-assistant/support#6851).
A zone or sub-device identifier type is needed because chassis MAC plus IP are not enough on their
own (NAD CI580 with four BluOS zones, music-assistant/support#6847; MusicCast zone 2). A differing strong
identifier (UUID, CAST_UUID, AIRPLAY_ID, serial) vetoes a match before any MAC or IP fallback;
today only AIRPLAY_ID does (several shairport-sync receivers on one host, music-assistant/support#6807, music-assistant/support#5985). Two
native protocols for one device need a priority: vendor native over hass_players over generic
(music-assistant/support#6836, music-assistant/support#6468).

**Roles are not identity.** *(review)* Roles come from heuristics that flip (`is_google_device`,
the Apple vendor check, the web-player product names, Cast identification failing, Sendspin's
`restore_bridge_identity`). A role change of the same protocol id is a re-attach with another role,
never a second player or a new id.

**Lifecycle.**
- `register` / `register_or_update` = attach (or re-attach) a protocol. A non-permanent
  `unregister` = detach: the player stays, turns unavailable (or "available, powered off" under an
  external power control), keeps its queue. Permanent unregister = remove.
- Detaching releases the queue's audio and pending next-track work (today's `on_player_remove`
  part from music-assistant/server#7268) without dropping the queue.
- Plugins that use `PLAYER_ADDED` as "device back" or `get_player() is None` as "device gone"
  (Spotify Connect, AirPlay Receiver, Plex Connect, Yandex, Music Quiz, party, Cast Sendspin
  bridge) switch to the availability change that `PLAYER_UPDATED` already carries.
- Provider lookups of their own objects go through the provider (`self.get_endpoint(id)`,
  `self.endpoints`, renamed per decision 4), never `mass.players.get_player()`. About 185 call
  sites, mechanical except group-member and leader lookups (Sonos, Squeezelite, Snapcast,
  MusicCast, AirPlay, WiiM, sync_group) that need a per-site look.
- *(review)* Re-attach hook: the protocol re-initialises transport and subscriptions (HEOS and DLNA
  after an AVR power cycle music-assistant/support#5565, Sonos after a reboot music-assistant/support#5792, Sendspin after
  `goodbye shutdown` music-assistant/support#6888); the player restores group membership; the queue keeps its
  resume position; whether playback resumes by itself is decided per trigger (music-assistant/support#6191).
- *(review)* Derived protocols (Sendspin over Cast or AirPlay) are built from the player's protocol
  list at boot, not from runtime provider events (music-assistant/support#6192: Cast players without bridges for a
  whole session because of load order).
- *(review)* A command to an unavailable player raises `PlayerUnavailableError`, except power on,
  and play on a player with an external power control (music-assistant/server#7435). Today it
  returns success and does nothing (music-assistant/support#6493). A plugin holding a source for an offline device
  starts it on the became-available transition (music-assistant/support#7017).
- *(review)* Session-only protocols (decision 2) get no persisted config unless the user customised
  them, are purged on detach otherwise, are not offered in plugin pickers, and a second attach of
  the same protocol id replaces the first (music-assistant/support#6773, music-assistant/support#6822, music-assistant/support#6064).
- *(review)* Every player persists `last_seen`; `PlayerState` exposes `unavailable_since`; the
  settings page gets "remove all unavailable players". Auto-forget stays out of scope.

**What goes.** The universal player provider, the universal create/merge/takeover paths and the
two-sided link bookkeeping in `protocol_linking.py` (about 1,150-1,250 of 3,290 lines),
`ProtocolBackedPlayer` as a provider base (its behaviour moves into the player), the 45 s "wait for
cached parent" reservation and `_get_online_player`'s "registered again" check. What stays:
identifier matching, derived (bridge) protocols, output selection, control resolution and protocol
grouping (about 1,500 lines).

## Decisions (2026-11-09)

1. **Delivery**: one release. Phase PRs target a long-lived `feature/player-split` branch off dev,
   which merges dev in regularly and lands on dev as a whole. No CI change: `test.yml` runs by
   hand (`workflow_dispatch` with `ref`) on the feature branch.
2. **Which configs come back at startup**: every enabled player config of a loaded provider,
   except session-only players (Sendspin browser/app and virtual players, displays, visualizers,
   lights, line-in sources), which keep "exists while attached". A one-time migration marks the
   existing Sendspin standalone configs; from then on the protocol reports it. Stale devices show
   as unavailable until the user removes them.
   *(review: proposed change)* Every enabled player comes back at boot regardless of which
   providers are loaded; a player whose protocols' providers are all unloaded is unavailable.
3. **A native protocol for a device that already has a player** attaches to it; the id does not
   change. The universal takeover path goes.
4. **Naming**: the core object is `Player` (what `mass.players.get_player()` returns); providers
   subclass `PlayerEndpoint` and find their own objects through the provider
   (`get_endpoint(id)`, `endpoints`). The MSX Bridge upstream repo needs the same rename.
   *(review: proposed change)* Naming as at the top of this document: rename `PlayerEndpoint` to
   `PlayerProtocol` and `get_endpoint` / `endpoints` (provider and controller) to `get_protocol` /
   `protocols` before Phase 1; the MSX Bridge upstream repo follows.
5. **Ids**: existing ids are frozen as player ids; new players get MA-minted ids (see above).
   *(review: proposed change)* Every player also carries a legacy id (see Legacy id).

Additional decisions from the review *(review)*:
- Rollback of the config migration is not a requirement.
- A new player that is disabled by default is registered on the branch for now (see Phase 0
  state). The end state of #415, "no player until the user enables it", restores the early return
  in `register()` and gets a test.

Defaults for the smaller questions:
- `PlayerState.provider` for a protocol-only player stays `"universal_player"` until the frontend
  no longer needs it (the frontend's delete check and settings link read it).
  *(review: proposed change)* Dropped. The frontend source has no reference to
  `universal_player`. Clients need `player.provider` to resolve to a loaded provider instance: the
  frontend reads `REMOVE_PLAYER` from `api.getProvider(player.provider)`, the HA entity calls
  `provider.name` on `mass.get_provider(player.provider)` when the manufacturer is empty
  (Chromecast reports an empty one), and the server routes enable/disable through
  `mass.get_provider(config.provider)`. A literal breaks all three once the provider is gone. See
  Ownership.
- Two players that turn out to be one device are still merged (older wins), as today.
  *(review: proposed change)* Only while the younger one is unused, otherwise manual. See First
  sight and merge.
- Last known device info and identifiers are stored lazily on the player's config at point of use,
  no extra provider requests.
- Providers that can power on an offline device themselves (Sonos Wake-on-LAN) get a protocol hook
  later; external power controls cover the rest.

## Phases (PRs into `feature/player-split`)

0. **Preparation, no behaviour change** (#413, #414; several small PRs):
   - provider-scoped protocol lookup, and all providers moved to it (still returns today's object);
   - plugins move to availability transitions;
   - `Player.__init__` stops writing config (registration creates it).

   *(review)* See Phase 0 state below.
1. **Player from config + attach/detach for native protocols** (#415). The registry holds players,
   provider objects become `PlayerProtocol`s; boot from config per decision 2; the one-time id
   migration; the player keeps the last known device info and identifiers (nothing reads them
   before this phase); detach instead of unregister; queue lifetime follows the player. Fixes
   Squeezelite on a plug and every native player after a restart.

   *(review)* Also: the config shape decision and the record split if chosen; legacy id and
   `last_seen`; ownership (provider-independent boot, controller-handled enable/disable/remove);
   the three layers (pure state derivation); the aggregation rules; the re-attach hook; command
   semantics while unavailable; session-only protocols without persisted config; the
   disabled-by-default early return.
2. **Audio protocols on players, universal player provider removed** (#416). Universal configs
   become players with the same `up…` ids (idempotent migration that never raises: identify them by
   `provider == "universal_player"`, never by type); takeover becomes attach.

   *(review)* Also: protocol re-identification; the collect window and merge policy; the
   identifier veto and the zone identifier; claimed extra instances; native priority; derived
   protocols from config; the migration repairing edges.
3. **Cleanup** (#417). `ProtocolBackedPlayer` folded into `Player`, sync_group onto a player
   "protocol for domain X" API, test suites rewritten around attach/detach
   (`test_protocol_linking.py` is 12.7k lines, 28 test files poke `_players`).

   *(review)* `_players` is touched by 96 test files on dev today, not 28. The scenario suite below
   is the acceptance gate.
4. **Later, after the release** (#418): move existing players to MA-minted ids together with a
   migration in the Home Assistant integration (HA otherwise drops and recreates the entities).

   *(review)* The HA migration maps on `legacy_id` with `async_migrate_entries` for `unique_id` and
   the device identifier; the server keeps accepting legacy ids.

A shutdown bug found while researching this (queue state of the last 5 s lost because providers
unload before the queue controller flushes) is independent and goes to dev as its own fix.

## Phase 0 state on `feature/player-split` *(review)*

Reviewed at head `9b7b44fda`; the notes in brackets were checked against that head afterwards.

- A new player that is disabled by default is fully registered on first discovery: `register()`
  creates the config with `enabled=False` and then checks `player.state.enabled`, which comes from
  the state built in `Player.__init__` before any stored config exists and therefore defaults to
  `True`. Effects: an AirPlay Mac enters the registry and gets a universal player after the 15 s
  evaluation, a Cast TV claims its AirPlay protocol, and `PLAYER_ADDED` and a queue are created for
  a disabled player. [This was already so before Phase 0. A skip was tried in
  music-assistant/server#7452 and reverted: the stored config of a protocol device is of protocol
  type and the settings list hides those, so the user could not enable it. See the additional
  decisions.]
- `get_available_player` in `helpers/player.py` ignores `enabled`; `PlayerAvailabilityTracker`
  seeds from `iter_players(return_unavailable=False)` but its per-event check uses a different
  filter. Align both on `state.enabled`. [Checked: `state.available` already includes `enabled`
  (`models/player.py`, `available=self.enabled and (…)`), so both apply the same filter; nothing to
  align.]
- The provider lookup (`get_endpoint`, to be renamed `get_protocol`) is `mass.players.get_player()`
  without an ownership check, so the ~145 converted call sites were never validated against foreign
  ids. Add a warning when `protocol.provider is not self` so mismatches surface before Phase 1
  turns them into `None`.
- Still open from Phase 0: Music Quiz uses `get_player() is None`; the sync_group, universal_group
  and universal_player lookups are deferred; the last known device info and identifiers that
  #413's text claims are not stored by any PR (fix the issue text or do it). [Music Quiz and party
  keep `get_player() is None` on purpose in music-assistant/server#7449: after the split it means
  the player was removed, which is when a session should end. The device info moved to #415; the
  issue text still needs the fix.]
- The branch is behind `dev` (12 commits at review, 14 now; the merge is clean;
  music-assistant/server#7446 changes the power-control semantics Phase 1 relies on). "pytest
  passes" is unticked in all three PRs. [Each PR and the combined branch passed a full `test.yml`
  run by hand; the box was left unticked.]

## Migration *(review)*

- Back up `settings.json` before the first run of the split migration.
- Validate and repair edges, not only move keys: orphan protocols without a parent, two instances
  of one domain on one parent, parents holding another device's identifiers and `cached_arp_mac`,
  a parent edge pointing at the wrong zone or at a native player that never registers again
  (music-assistant/support#6807, music-assistant/support#6847, music-assistant/support#6461, music-assistant/support#6737).
- Collect the `settings.json` files from those issues as migration test fixtures. Ship a
  diagnostics dump of the resulting topology (player → protocols with roles and availability).
- Publish players before providers load, so HA receives one consistent list at boot.

## API and models *(review)*

- `PlayerState` gains `protocols` (domain, provider, role, available, active), `legacy_id` and
  `last_seen` / `unavailable_since`; `output_protocols` stays as the audio-capable subset.
- `PlayerType.PROTOCOL` stays in the enum for client compatibility but is no longer emitted.
- Event semantics: `PLAYER_ADDED` on first sight only, `PLAYER_REMOVED` on user removal only,
  attach and detach as `PLAYER_UPDATED`. The HA integration already handles that. The protocol
  players' own HA entities (`…_2`) disappear; some users control those directly (music-assistant/support#6293),
  so this needs a release note.
- The frontend already hides unavailable players from the main list; settings and HA show them.

## Scenario suite *(review)*

Attach/detach acceptance tests, written before Phase 1 and used as the gate for phases 1 and 2:
several AirPlay receivers on one IP; a multi-zone chassis behind one MAC and IP; a room of three
DLNA renderers; a device re-flashed to another firmware with the same MAC; a provider removed while
audio protocols remain; the native protocol drops while audio protocols stay available; a role
flip of the same protocol id; a disabled-by-default device; a session-only web player; a merge of
an unused duplicate; a universal player config restored with the same `up…` id; power on of an
offline device through its external power control; commands to an unavailable player.

## Risks

- Size: 27 provider packages (15 S, 7 M, 4 L: AirPlay, Chromecast, Sendspin, sync_group), plus the
  linking tests. Phase 0 makes the later phases mostly a change in the players controller.
- Clients see more players: unavailable players stay listed during provider reloads and after a
  restart. Intended, but the frontend and HA will show it.
- Daemon-per-player plugins (Spotify Connect, AirPlay Receiver) would start for offline players at
  boot unless they check availability.
- Config migrations run every start before providers exist and abort startup on an exception:
  every new step needs a guard and must be shape-idempotent.
- *(review)* After music-assistant/server#7449 the daemon plugins (Spotify Connect, AirPlay
  Receiver) stop advertising an unavailable-but-wakeable device (Sonos sleep, Raumfeld standby)
  that dev still advertised; check against music-assistant/server#7435 and music-assistant/server#7446 before offline
  players stay registered.
- *(review)* With identities in config, a corrupt `settings.json` (music-assistant/support#5716) loses every player
  id, not only settings; the backup and the legacy id limit the damage.

## Out of scope

Frontend and models changes beyond what compatibility needs, an auto-forget for stale players,
offline power hooks per provider.
