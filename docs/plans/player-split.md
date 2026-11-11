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
*(decided 2026-11-09)* The design keeps the door open for a player composed of several protocol
players (a stereo pair or surround set of several devices): `Player.protocols` is a collection with
roles and the derivation runs over all of them; "at most one native" is an attach rule, not a
schema assumption.

A protocol attaches to exactly one player. One persisted edge records it: the protocol's
`protocol_parent_id` (already the canonical side today; the key name stays to avoid a data move),
stored on every protocol whose id differs from its player's id. Protocol configs stay where they
are (`players/{protocol_id}`). At startup each player builds its protocol registry from the
protocols that point to it, so a protocol belongs to one player by construction. The parent-side
`linked_protocol_ids` list is no longer read or written.
*(decided 2026-11-09)* The edge is `player_id` on the protocol record under
`player_protocols/{protocol_id}`, replacing `protocol_parent_id`; protocol configs move out of
`players/` (see Config shape).

**Player id.** Owned by the player, stable for its whole life, never derived again and never
switched by a protocol. A one-time migration freezes the current visible id of every existing
player (RINCON_…, Cast uuid, `up…`) as its player id, so HA entities, user filters, groups, DSP,
queues and plugin bindings stay as they are. A player created after the change gets an
MA-minted opaque id. Moving existing players to MA-minted ids, with a clean HA migration, is a
later step.

**Legacy id.** *(review)* Every player gets a `legacy_id`: the id it had before the split (for a
frozen id the id itself; after #418 the pre-move id), plus any id absorbed by a merge, together with
the absorbed player's own legacy ids. The API resolves a legacy id wherever a player or queue id is
taken. `PlayerState` exposes it. The HA migration of #418 maps `unique_id` and the `(DOMAIN,
player_id)` device identifier on it.
*(decided 2026-11-09)* The field is `legacy_ids: list[str]`, empty by default, on the models
`Player` (one additive models release, safe for old clients and the pinned HA models version),
exposed on `PlayerState` and persisted on the player record. One resolver (a reverse index of
legacy id to player id) serves two entry points: the `api_command` dispatch rewrites the player id
and queue id arguments before the handler is called, and `get_player` applies it for internal
callers. Legacy ids are accepted indefinitely. #418's HA migration maps `unique_id` and the device identifier on them.

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
*(decided 2026-11-09)* The players controller handles enable, disable and remove itself, and every
player is removable (today only the six providers that declare `REMOVE_PLAYER` can remove one; the
other 21 of 27 cannot). The six `remove_player` overrides become one optional per-protocol hook.
`PlayerState.provider` is the primary protocol's loaded provider instance as above; when nothing
is attached it keeps the last stored value, so it does not always resolve to a loaded provider.
Removing a provider deletes that provider's protocol records; the player is removed only when none
remain. With it: the removal sweep in `config/providers.py`, `iter_players(provider_filter)`,
`PlayerProvider.protocols` and the provider unload loop filter on protocol records instead of
players, and the config translation owner of a player record follows the primary protocol.
*(review)* The frontend gates on the provider too: `src/helpers/player_settings_actions.ts` hides
enable/disable when `api.getProvider(config.provider)` is absent, and `playerCanBeDeleted` requires
a loaded provider advertising `REMOVE_PLAYER` or `REMOVE_GROUP_PLAYER`. A frontend change in the
same release gates both on the player state instead, or restored players with unloaded providers
can neither be disabled nor deleted in settings.
*(decided 2026-11-09)* Every player provider advertises `REMOVE_PLAYER`. The frontend companion PR
gates settings, enable/disable and delete (groups included) on the stored player config, with the
live player state optional (a disabled player has no live object), and is a landing dependency of
the cutover.

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
*(decided 2026-11-09)* The split as proposed: `players/{player_id}` is the player record only,
protocol instances move to `player_protocols/{protocol_id}` with `player_id` as the edge, replacing
`protocol_parent_id`. One idempotent migration that tolerates malformed records, with
`settings.json` backed up first (failure handling under Migration). The `config/players/*` API wire
format stays unchanged for now: the `||protocol||` virtual keys are still merged on read and
stripped on write, so the frontend is untouched.

**State of a player with several protocols.** *(review)*
- `available` = any attached protocol `available_for_playback`; `needs_setup` is aggregated the
  same way. Today only `ProtocolBackedPlayer` does this; a native parent goes unavailable when its
  native protocol drops although AirPlay and Cast still work (music-assistant/support#6951).
  *(decided 2026-11-09)* Two predicates, as today: the published `available` also holds for a
  device switched off at its external power control (see Lifecycle), `available_for_playback` does
  not.
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
*(decided 2026-11-09)* "Unused" is three persisted signals: no custom config, no saved queue (never
played), no group membership; "created this session" is not a signal. Otherwise both players stay
and a new `players/merge` API command (Phase 2) merges them by hand; the UI action is a frontend
follow-up. The absorbed id becomes a legacy id. The shared-domain veto of today's merge check stays.
*(decided 2026-11-09)* The collect window stays only for the first sight of an audio-only unknown
device and ends early once the identifiers needed for matching are in (max 15 s); native protocols
mint at once, known devices never wait. "Unused" in detail: no custom config (user-editable keys
only, never automatic metadata), no saved queue items, no group membership. Auto-merge is a
separate PR.

**Where one protocol per domain is not enough.** *(review)* A native protocol can claim further
instances of one domain, attached hidden (a Raumfeld room of three DLNA renderers, music-assistant/support#6851).
A zone or sub-device identifier type is needed because chassis MAC plus IP are not enough on their
own (NAD CI580 with four BluOS zones, music-assistant/support#6847; MusicCast zone 2). A differing strong
identifier vetoes a match before any MAC or IP fallback: CAST_UUID, AIRPLAY_ID and the zone
between any two protocols, UUID and serial only when both values were reported by a protocol of
the same domain (the Cast and DLNA sides of one device each report a UUID of their own); a veto
applies to the merged identity of a player;
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
- *(decided 2026-11-10)* Access to a grouped player. With access to a player a user may take it out
  of its group and use it alone: ungroup, power off, Play Media when the player's "play media
  overrides group" setting is on, select source. Anything that plays on, or steers, the group needs
  access to the group player or the leader. Adding to a group needs access to the target as named
  and to every player added; removing needs access to the target (including the group player a
  command is passed on to) or to every player removed. `player_queues/play_media` takes the queue id
  of the player the user picked, and the server decides where the request belongs: add, next and
  replace-next go to the queue the player is playing for; play and replace go to the player alone or
  to the owner's queue, by that setting.
- *(decided 2026-11-11)* Removing a player whose device is still on the network resets it: after
  the removal the controller asks the player's provider to discover again, so the device is found
  again as a new player with default settings. A player that should stay away is disabled, not
  removed. A removed group player is gone. The removal hook also reaches a protocol whose player is
  not live (disabled, or held by its provider without being attached), since providers with a
  configured device list or a registry of their own depend on it.
- *(review)* Session-only protocols (decision 2) get no persisted config unless the user customised
  them, are purged on detach otherwise, are not offered in plugin pickers, and a second attach of
  the same protocol id replaces the first (music-assistant/support#6773, music-assistant/support#6822, music-assistant/support#6064).
  *(decided 2026-11-09)* A session-only player that detaches emits `PLAYER_REMOVED`; its record
  stays if customised.
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
   *(decided 2026-11-09)* The proposed change is decided. Such a player is listed as unavailable in
   the settings list and in `players/all`, so HA sees one consistent list. The session-only marker
   migration (Sendspin browser/app players and the other session-only types) runs before this boot
   step. A record of a provider that is not installed shows a "provider not installed" reason in
   the UI.
   *(decided 2026-11-09)* The session-only marker migration deletes uncustomised Sendspin
   standalone records and marks customised ones; customised = user-editable keys, never automatic
   metadata.
   *(review)* Prerequisite, in Phase 1: the Home Assistant entity reads
   `device_info.manufacturer or provider.name` (`entity.py`), so a restored player with an unloaded
   provider and an empty manufacturer would raise instead of becoming an unavailable entity. The
   server never publishes an empty manufacturer for such a player (filled from the stored device
   info or the stored provider name) before these players are published; an HA-side guard is a
   follow-up, not part of #418.
3. **A native protocol for a device that already has a player** attaches to it; the id does not
   change. The universal takeover path goes.
4. **Naming**: the core object is `Player` (what `mass.players.get_player()` returns); providers
   subclass `PlayerEndpoint` and find their own objects through the provider
   (`get_endpoint(id)`, `endpoints`). The MSX Bridge upstream repo needs the same rename.
   *(review: proposed change)* Naming as at the top of this document: rename `PlayerEndpoint` to
   `PlayerProtocol` and `get_endpoint` / `endpoints` (provider and controller) to `get_protocol` /
   `protocols` before Phase 1; the MSX Bridge upstream repo follows.
   *(done: music-assistant/server#7463)*
   *(decided 2026-11-09)* The class name stays `PlayerProtocol`; its id attribute becomes
   `protocol_id` at the cutover.
5. **Ids**: existing ids are frozen as player ids; new players get MA-minted ids (see above).
   *(review: proposed change)* Every player also carries a legacy id (see Legacy id).
   *(decided 2026-11-09)* As `legacy_ids: list[str]`, accepted indefinitely (see Legacy id).

Additional decisions from the review *(review)*:
- Rollback of the config migration is not a requirement.
- A new player that is disabled by default is registered on the branch for now (see Phase 0
  state). The end state of #415, "no player until the user enables it", restores the early return
  in `register()` and gets a test.
- *(review)* Branch rule until Phase 3: delete and replace yes, move and rename no. Moving files and
  renaming on a long-lived branch that merges `dev` regularly creates a conflict on every `dev` PR
  that touches the same files. Structure work (bucket B of the cleanup) waits for Phase 3 or right
  after landing; the `PlayerProtocol` rename was the one exception (music-assistant/server#7463).
- *(review)* The ten migration steps marked "remove after 2.10" (2.10.9 is released) are removed in
  their own PR into `dev` now, not on the feature branch. A user who upgrades from 2.9.x straight to
  that release skips them.
- *(review)* Review rule after the cleanup: the one-time cleanup removes the complexity and
  fragility; from then on every change to the players core gets a critical review by an AI model
  against the README scenarios and by a member of the core team. No CODEOWNERS or PR template
  mechanics; the README and that review are the guardrail.

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
  *(decided 2026-11-09)* Dropped. The Phase 2 migration rewrites the stored `provider` of `up…`
  records to the primary protocol's instance, because the stored field drives the translation
  owner, the `config/players?provider=` filter and the removal sweep. `is_native_player` becomes
  role based.
- Two players that turn out to be one device are still merged (older wins), as today.
  *(review: proposed change)* Only while the younger one is unused, otherwise manual. See First
  sight and merge.
  *(decided 2026-11-09)* As proposed, with "unused" defined in First sight and merge.
- Last known device info and identifiers are stored lazily on the player's config at point of use,
  no extra provider requests.
- Providers that can power on an offline device themselves (Sonos Wake-on-LAN) get a protocol hook
  later; external power controls cover the rest.

## Phases (PRs into `feature/player-split`)

*(decided 2026-11-09)* Phases 1-3 are implemented as ONE continuous PR sequence to the end state,
without scaffolding: no intermediate class names, no shims, no bridge for the universal provider.
The cutover is one PR, reviewable per commit. The sequence:
[player-split-pr-sequence.md](player-split-pr-sequence.md).

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

   *(review, cleanup)* Delete what the change makes obsolete, on the code it rewrites:
   - Attach and detach replace `register`, `register_or_update` and `unregister`. Swapping in a
     replacement instance becomes a detach plus an attach, so the swap logic in
     `register_or_update` (copying config onto the new instance, re-marking it initialized, the
     role-change cleanup) goes, and so do the guards that exist only because registration was
     creation: `_registration_aborted`, the second `_teardown_in_progress` check inside the lock,
     the "not the registered instance" guard in `signal_player_state_update`, the queue re-create
     race after an aborted registration, and `_get_online_player`'s "registered again" check.
   - Enable, disable and remove are controller concerns: drop the default behaviour of
     `PlayerProvider.on_player_enabled` (re-run discovery after 5 s) and `on_player_disabled`
     (unregister); a provider gets an optional hook per protocol if it needs one.
   - If the record split is chosen, protocol configs become real records: the virtual
     protocol-prefixed and plugin-prefixed config entries (`CONF_PROTOCOL_KEY_SPLITTER`,
     `CONF_PLUGIN_KEY_SPLITTER`, merged on every read and stripped on every write) become a
     sub-config returned next to the player config. That removes about a third of
     `config/players.py` (1,379 lines on `dev`).
     *(decided 2026-11-09)* The record split is chosen, but the wire format stays: the virtual keys
     keep being merged on read and stripped on write. Returning protocol settings as a sub-config
     and removing that layer (about 500 lines of `config/players.py`) is a frontend-coordinated
     follow-up. Sequencing: attach/detach and the frozen-id boot first; the tree split is the last
     Phase 1 PR, with a read-through shim for the universal provider and the linking layer that
     still read `players/{protocol_id}` until Phase 2 removes them (or the split moves to the first
     Phase 2 PR if the shim is not worth it).
     *(decided 2026-11-09)* No read-through shim: the tree split lands in the cutover PR together
     with the removal of the universal provider and the linking layer.
   - Events: write the semantics down (`PLAYER_ADDED` on first sight, `PLAYER_REMOVED` on user
     removal, `PLAYER_UPDATED` for attach and detach) and plan the removal of the compatibility
     `PLAYER_CONFIG_UPDATED` emitted on feature changes for the HA integration, together with the
     HA release that re-evaluates features on `PLAYER_UPDATED` (see Phase 4).
     *(decided 2026-11-09)* Disable emits `PLAYER_REMOVED` instead of today's update: the object
     goes, the record stays, HA drops the device.
2. **Audio protocols on players, universal player provider removed** (#416). Universal configs
   become players with the same `up…` ids (idempotent migration: identify them by
   `provider == "universal_player"`, never by type); takeover becomes attach.

   *(review)* Also: protocol re-identification; the collect window and merge policy; the
   identifier veto and the zone identifier; claimed extra instances; native priority; derived
   protocols from config; the migration repairing edges.

   *(review, cleanup)* Delete what the change makes obsolete, on the code it rewrites:
   - One invariant instead of nine repairs: the edge lives on the protocol record and nowhere
     else, validated once in the split migration and checked once in the diagnostics dump. Delete
     the startup repair of parent links (`_repair_protocol_parent_links`), the weekly task that
     fixes sync group member configs (`_fix_group_member_configs`), the cache recovery of linked
     protocol ids (`_recover_cached_protocol_links`), the reconciliation of stored protocol ids in
     the universal provider (`_resolve_stored_protocol_ids`), the eviction of a protocol from
     other parents (`_evict_protocol_from_other_parents`), and the migrations for self-referential
     links, orphaned disabled protocol configs, AirPlay Receiver ghost players and the universal
     config absorption. Eight of the nine repair links or protocol configs; the ghost-player step
     cleans up identity churn. None has a reason to exist once the parent-side list is gone.
   - Roles are attributes, not types: delete every place that flips `PlayerType` on a provider
     object or heals it in config (`_cleanup_player_type_transition`,
     `_finish_player_type_transition`, `set_player_type`, Sendspin's `restore_bridge_identity` and
     its PROTOCOL/PLAYER switch, Chromecast's and Squeezelite's PROTOCOL-or-PLAYER decision). A
     `PlayerProtocol` reports a role; `PlayerType` on the `Player` is player, stereo pair, group,
     display, visualizer, light or source, never protocol. On `dev` providers set the type in 24
     places (most once per class; 7 are Sendspin's runtime switches), and Sendspin takes over
     another player's config in 3.
   - Identifier matching becomes one pure, table-driven function, with precedence and veto rules
     in one place and a test per rule. Identifiers become a set of (type, value) pairs instead of
     one value per type, so a device with two MACs (ARP and reported) is modelled rather than
     carried in `extra_data["reported_mac"]`. The DLNA provider normalises the Sonos `_MR` suffix,
     the matcher does not know it. Sibling matching (HEOS through its AirPlay child's identifiers)
     disappears once the `Player` merges identifiers from every protocol.
   - Provider-side takeovers go: Sendspin's `_take_over_mac_player`, the Raumfeld room UDN to
     serial move and the callers of `carry_over_player_config` are replaced by the one
     re-identification rule in the controller.
   - Migrations: every protocol-link repair step is removed when the split migration lands, since
     it supersedes them. `migrations.py` (2,109 lines on `dev`) stays a list of live steps, each
     with its removal release.
3. **Cleanup** (#417). `ProtocolBackedPlayer` folded into `Player`, sync_group onto a player
   "protocol for domain X" API, test suites rewritten around attach/detach
   (`test_protocol_linking.py` is 12.7k lines, 28 test files poke `_players`).

   *(review)* `_players` is touched by 96 test files on dev today, not 28. The scenario suite below
   is the acceptance gate.
   *(review: correction)* Checked on `dev`: 25 test files use the private `_players` registry, 8 of
   them through `players._players`. The 96 counted substring matches, which are mostly the public
   `all_players` and `iter_players`.

   *(review, cleanup)* Structure work, in Phase 3 or right after landing (branch rule above):
   - Split the players controller (5,524 lines on `dev`) by concern: registry and lifecycle,
     commands, volume, power, sleep timer, grouping, announcements and audio sources are already
     mixins or sections; make them modules with a narrow interface to the registry. Split
     `protocol_linking.py` (3,379 lines) into matching, output selection and grouping translation.
   - Grouping as one plan and one executor. Group membership and leader lifecycle are 19 of the 69
     relevant support issues, and the code is a chain of `_try_group_via_*` fallbacks: about 1,040
     lines of grouping translation in `protocol_linking.py` plus about 1,100 in the controller's
     grouping and `set_members` paths. Replace them with a pure `plan_group(leader, members) ->
     GroupPlan` (which protocol carries each member, who leads, what has to be released) and a
     single executor that applies a plan. The plan is unit-testable against every scenario without
     a device.
   - Locks: a command that cannot take the player lock within 30 seconds runs without it today
     (`PLAYER_LOCK_TIMEOUT`); only the queue play actions and the next-track handover take it
     strictly (up to 120 s, then `ResourceBusyError`). Make the timeout fail loudly and give the
     provider calls it papers over their own timeouts; a silent lock bypass is how two commands end
     up on one queue.
   - Timers as state machines: the 5 s clear of the remembered source when idle, the 5 s external
     source takeover check, the external power off stop delay, the 2 s debounced update of all
     players, the 0.5 s queue update delay and the 15/45 s protocol evaluation delays. Keep the
     ones that survive the split as named constants with the reason next to them; move the ones
     that encode state (source memory, takeover detection, power on in progress) into explicit
     per-player state with tests.
   - Model compatibility aliases: the models `Player` (1.1.236) writes 9 compatibility keys on
     serialisation (`display_name`, `state`, `group_childs`, `extra_data`, `hide_player_in_ui`,
     `device_info.mac_address`, `device_info.ip_address`, the boolean `powered` marked TEMP
     2025-03-15, and `group_volume = 0`) and reads 4 back on deserialisation. Pick a minimum client
     version per consumer (HA integration, python client, mobile app, frontend), announce it, and
     drop them.
     *(decided 2026-11-09)* `group_childs`, `extra_data`, `hide_player_in_ui`,
     `device_info.mac_address` and `device_info.ip_address` are dropped in models#463. The others
     stay until the mobile app reads `name` / `playback_state` and HA treats `powered=None` as on
     (HA >= 2026.5.0 for `group_volume`).
   - Complexity suppressions: the 10 `noqa PLR09xx/C901` sites in the players, queues and sync
     group code (12 with `config/players.py` and `migrations.py`; set_members handling, output
     protocol config entries and the migration runner among them) are the hotspots. The splits
     above should remove the need for each, not keep the marker.
   - Test harness: a fake provider and a scenario DSL (attach, detach, report state, assert state
     and events), so tests go through the public API instead of `_players`;
     `test_protocol_linking.py` is 12,808 lines. The DSL itself comes first, before Phase 1,
     because the scenario suite is written in it (see Scenario suite); moving the existing tests
     onto it is the Phase 3 part.
   - Documentation as a contract: rewrite the players README as the model spec, covering the three
     layers, the roles, the invariants (one edge, one native, one per domain, roles are not
     identity), the matching table, the event semantics, and every scenario of the suite worked
     out end to end (what attaches, what the state becomes, which events go out, what the user
     sees), so the README and the tests describe the same cases. The README stays the single place
     a contributor or a model reads before touching the players core.
4. **Later, after the release** (#418): move existing players to MA-minted ids together with a
   migration in the Home Assistant integration (HA otherwise drops and recreates the entities).

   *(review)* The HA migration maps on `legacy_id` with `async_migrate_entries` for `unique_id` and
   the device identifier; the server keeps accepting legacy ids.
   *(decided 2026-11-09)* The field is `legacy_ids` (a list); legacy ids are accepted indefinitely.

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
  *(decided 2026-11-10)* A record the migration cannot convert is logged and skipped. When the
  backup cannot be written or a whole step fails, startup stops with a clear error, nothing is
  saved, and the next boot retries.
- Validate and repair edges, not only move keys: orphan protocols without a parent, two instances
  of one domain on one parent, parents holding another device's identifiers and `cached_arp_mac`,
  a parent edge pointing at the wrong zone or at a native player that never registers again
  (music-assistant/support#6807, music-assistant/support#6847, music-assistant/support#6461, music-assistant/support#6737).
  *(decided 2026-11-10)* A stored UUID that ends in `_MR` (the UDN of a Sonos renderer) loses the
  suffix, as a newly reported one does, so the Sonos player still matches it by UUID.
- Collect the `settings.json` files from those issues as migration test fixtures. Ship a
  diagnostics dump of the resulting topology (player → protocols with roles and availability).
- Publish players before providers load, so HA receives one consistent list at boot.

## API and models *(review)*

- `PlayerState` gains `protocols` (domain, provider, role, available, active), `legacy_id` and
  `last_seen` / `unavailable_since`; `output_protocols` stays as the audio-capable subset.
  *(decided 2026-11-09)* `legacy_ids: list[str]` instead of `legacy_id`.
- `PlayerType.PROTOCOL` stays in the enum for client compatibility but is no longer emitted.
- Event semantics: `PLAYER_ADDED` on first sight only, `PLAYER_REMOVED` on user removal only,
  attach and detach as `PLAYER_UPDATED`. The HA integration already handles that. The protocol
  players' own HA entities (`…_2`) disappear; some users control those directly (music-assistant/support#6293),
  so this needs a release note.
  *(decided 2026-11-09)* `PLAYER_REMOVED` also on disable and on the detach of a session-only
  player (see Lifecycle and Phase 1 events).
- The frontend already hides unavailable players from the main list; settings and HA show them.

## Scenario suite *(review)*

*(review, cleanup)* Written in the scenario DSL of the test harness (Phase 3 list), so the fake
provider and the DSL come first.

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
