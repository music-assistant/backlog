# Player split: PR sequence to the end state

Approved by the maintainer on 2026-11-09; the implementers' spec for the PRs into
`feature/player-split`. The plan [player-split.md](player-split.md) is the decision record and wins
where they disagree.

Facts checked on `feature/player-split` at dc5d1646e; scenario suite at origin/player-split/scenarios
56fc3cdb5 (second fix round landed: the AirPlay veto case passes, the "played" duplicate holds a
saved queue; 26 strict xfail cases: 15 "phase 1", 11 "phase 2"). PR 1 and PR 3 landed at 0852b2534;
their sections describe what landed. Paths relative to `music_assistant/` unless they start with
`tests/`. Binding rules applied: no step a later PR throws away; phases 1-3 as one sequence on
`feature/player-split`; every PR leaves both suites green with the remaining markers strict;
file moves and renames only in the tail; the cutover is one PR reviewable per commit; auto-merge is
separate; disable emits `PLAYER_REMOVED`; class name `PlayerProtocol`, id attribute `protocol_id` from
the cutover on; a `Player` may later hold several protocol players, so nothing below assumes one.

Numbering keeps v1's labels so the reviews stay readable: PR 6 and PR 7 of v1 fold into PR 4 (§6a);
a new PR 3b carries the permanent layers that can land before the cutover.

## 1. End-state shapes

**Player** (`models/player.py`, controller-owned, what `get_player()` returns). Owns `player_id`,
`legacy_ids`, `type` (never `PROTOCOL`), the record `players/{player_id}`, `protocols:
dict[protocol_id, PlayerProtocol]` with the attached role per entry, stored device data
(`default_name`, `device_info`, identifiers as a set of `(IdentifierType, value)`, `last_seen`), the
queue binding, transient state (today's `extra_data` flags, sleep timer, active output protocol,
active MA source, stop flag, external-pause bookkeeping, palette), control resolution, output
selection, `available_for_playback`, `offline_by_power_control`, `update_state()` with the
fingerprint diff and one `PLAYER_UPDATED`. It has no provider object: `provider_id` is the primary
protocol's instance (native, else highest-priority audio), else the stored `provider`. "At most one
native" and "at most one protocol per domain" are attach rules (§2), not fields: `protocols` is a plain
collection and every derivation rule iterates it, so a player built from two devices later needs new
attach rules, not a new shape. Today's core half (`models/player.py:1454` banner onward) is this
class; the `__final_*` work becomes the derivation module (PR 1).

**PlayerProtocol** (`models/player_protocol.py`, today an alias of `Player` at line 11). Today's
provider half (`models/player.py:413-1453`): `provider`, `protocol_id` (renamed from `player_id` in
PR 4, 424 `self.player_id` sites in 30 provider files plus the harness), `_attr_*` reported values and
the overridable properties, commands, `get_config_entries`, `handle_config_action`, `run_setup_flow`,
`on_config_updated`, `on_unload`, config access on its own record. New: `role` (reported: native,
control, audio, virtual), `session_only` (reported), `player: Player | None`, `on_attach(player)`,
`on_detach()`, optional `on_remove()`, `underlying_protocol_id`. `update_state()` stays the
provider-facing name and forwards to `player.update_state()` only while this object is the one
attached under its id (§6b). A protocol has no published state of its own; `self.state` reads in
providers follow the contract in §6c.

**Records.** `players/{player_id}`: base keys `player_id, provider, enabled, name, default_name,
player_type` plus values restricted to the player-level keys: the core default entries
(`config/players.py:776`: controls, volume limits and step, announce settings, `hide_in_ui`,
`expose_player_to_ha`, `icon`, `auto_play`, `idle_power_off`, `play_media_overrides_group`,
`preferred_output_protocol`) and the player-owned metadata `legacy_ids, last_seen, session_only,
device_identifiers, device_info, created_at`. `player_protocols/{protocol_id}`: `protocol_id,
provider, player_id` (the edge), `role` (informational cache, refreshed on every attach), `enabled`
(the output toggle), `default_name`, `values` (every key declared by the protocol's
`get_config_entries`, the per-output audio entries that `_get_player_config_entries` renders at
`config/players.py:716-775`, and the provider-written keys `cached_arp_mac`, `reported_mac`,
`underlying_player_id`, `protocol_experimental_note`, `sendspin_static_delay` and the like),
`setup_data`. Sync-group and universal-group `group_members`, `allowed_members` and `dynamic_members`
are declared by those protocols (`sync_group/player.py:331, 345`, read at 122, 307-308, 93;
`universal_group/player.py:197`) and therefore live on the group protocol's record (§6e). DSP and queue
settings keep `player_dsp/` and `player_queues/`. For a frozen id the two records share the id string,
never the key. Access is scoped (PR 3b): `get_player_value(player_id, key)`,
`get_protocol_value(protocol_id, key)` and their setters replace `get_raw_player_config_value` (27
files, 73 calls) and `set_raw_player_config_value` (25 calls in 9 files); saves route each wire key to
the record of the entry that declared it. The `config/players/*` wire format is unchanged:
`||protocol||` and `||plugin||` keys stay merged on read and stripped on write
(`config/players.py:233-255, 535-539`), native entries stay unprefixed.

**PlayerState** (models `Player`): today's fields plus `legacy_ids: list[str]`, `protocols:
list[PlayerProtocolInfo]` (protocol_id, domain, provider instance, role, available, active),
`last_seen`, `unavailable_since`. `type` never `PROTOCOL` (enum member stays); `output_protocols`
stays the audio-capable subset; `provider` as above, last stored value when nothing is attached.

**Pure state derivation** (PR 1). `derive_player_state(inputs) -> (PlayerState, effects)` in a new
module. Inputs: config-derived values; one snapshot per attached protocol (role, available,
available_for_playback, reported values, features, group topology in protocol ids, needs_setup and
reason); control states; the queue snapshot; peer snapshots (leader and members: state plus the raw
`powered` and `is_active_session` the group rules read); the audio-source session and its reported
media (`models/player.py:3159-3160`, `3287`); standing plugin sources and device inputs
(`3389-3419`); transient flags; the previous state and position anchors (`2795-2809`,
`2859-2861`); the clock. Effects returned, never performed inside: the media-updated callback
(`2232-2234`), the default-name persist (`2236-2237`), the position-jumped signal (`2245`), the
stop-called flag and the 5 s MA-source timer (`2863-2881`). Rules in PR 4, not PR 1: `available` and
`available_for_playback` as the two predicates of §6d (only the published `available` holds under an
external power control); `needs_setup` aggregated over the protocols; name = custom, else
native, else audio by priority, else stored `default_name`; power tri-state; control target per
feature = active output protocol, else native, else any available protocol; a control protocol
reporting itself a native follower clears the active output (today in
`wiim/linkplay_player.py:240-248`).

**Identity.** Frozen: every existing record keeps its id; `legacy_ids` starts empty. Minted: `pl` +
12 hex (`up` and snapcast's `ma_` are taken). Attach resolves identity as: stored edge of the
protocol's record, validated against the live veto rules of the player's attached protocols; else
identifier match against live players and the stored identifiers of disabled player records (natives
only against players without a native protocol record or with an offline one of the same provider
and domain, §6f; a match to a disabled player writes the edge and
creates no object, §2); else mint. Known devices never wait. Collect window: only an audio protocol that matches nothing and
has no valid hardware MAC yet (ARP pending, address missing) waits, at most today's 15 s
(`protocol_linking.py:439`), re-matching as identifiers arrive and minting as soon as a hardware MAC is
in or the window ends; a native protocol mints at once. Re-identification: a protocol matching a
player that has no attached protocol of its domain attaches to it; if the player holds an offline
protocol record of the same provider and domain, the newcomer takes over that record (values,
setup_data) under its own id, `preferred_output_protocol` is rewritten, the stale record goes. Merge
(PR 5): automatic only when the younger player is unused (§5, PR 5); otherwise both stay and
`players/merge` merges by hand. Absorbed id → keeper's `legacy_ids`, edges re-pointed, groups and
user filters rewritten, absorbed saved queue purged, `PLAYER_REMOVED(absorbed)`. The absorbed
player's own legacy ids go to the keeper with it.

## 2. Attach and detach

| Call | Case | Behaviour |
|---|---|---|
| `attach(protocol)` | id unknown | resolve identity; create the player from its record when not live; bind; `on_attach`; `PLAYER_UPDATED` (or `PLAYER_ADDED` + `QUEUE_ADDED` for a minted or session-only player). A disabled record gets no object: the record is created or updated, the provider keeps its object. |
| | same id, same object (Sendspin hello, `sendspin/provider.py:1843-1859`) | refresh: re-read reported values, `update_state`; a changed role = detach + attach on the same player; `on_config_updated` is not re-run. |
| | same id, other object (Squeezelite and Sendspin reconnect, Bluesound new IP) | replacement: `on_detach` on the old object without publishing, bind the new one. |
| `detach(protocol_id)` | temporary loss, provider unload, disable | player stays; `on_detach`; queue quiescing only if this protocol was the playback path (§6d); `unavailable_since` set when no protocol is left available; `PLAYER_UPDATED`. A session-only player is removed (`PLAYER_REMOVED`); a customised record stays for the next attach. |
| `remove_protocol(protocol_id)` | provider forgets a device (`chromecast/provider.py:316`, `sendspin/bridge_manager.py:178-184`, `chromecast/sendspin_bridge.py:906-910`, `sendspin/provider.py:780-784`) | detach + delete the protocol record; the player goes when no protocol record remains. |
| `remove(player_id)` | user, or provider removal leaving no record | detach all, `on_remove` per protocol, delete player and protocol records, DSP, queue settings, saved queue, group memberships, user filters; `PLAYER_REMOVED`. Every player is removable. |

Attach rules: at most one native per player (a second native stays separate until PR 8); at most one
protocol per domain (`_parent_has_active_protocol_from_domain`, `protocol_linking.py:1400`, kept);
role is fixed at attach (type flips at `sendspin/player.py:1508, 2176`, `sendspin/provider.py:1339`,
`chromecast/player.py:619-626`, `squeezelite/player.py`, `airplay/player.py`, `dlna/player.py`
become role reports). Callers move in PR 4 without a shim: `players.register` 11 sites in 9 files,
`register_or_update` 24 in 20, `unregister` 31 in 22 (positional `permanent` at
`hass_players/provider.py:148`, `fully_kiosk/provider.py:120`, `mpd/provider.py:107`,
`sync_group/provider.py:77`, `universal_group/provider.py:61`, `samsung_wam/provider.py:74`),
controller-internal `protocol_linking.py:1396` and `controller.py:1883`, the harness
(`tests/scenarios/harness.py:360, 364, 502, 513`).

## 3. Boot sequence and events

1. `ConfigController._load` → `migrate()` (`config/controller.py:318-329`; only file-not-found and
   JSON errors are caught, a raising step aborts startup). The player-split steps run last, after
   every older step, each behind its own run-once top-level flag (the pattern of
   `retired_local_audio_cleaned`, `constants.py:132-133`) and shape-tolerant: a record that cannot
   be converted is logged and skipped, and the step carries on with the rest. A step that fails as
   a whole is not caught: nothing is saved, its flag stays unset, startup stops with a clear error
   and the next boot retries, as for a failed backup. Backup first (§6i).
2. Core controllers (`mass.py:322-333`), then `post_setup` in order cache, tasks, streams, music,
   metadata, players, queues (`335-344`). `players.post_setup` boot step: a `Player` for every
   player record that is enabled and not session-only; `set_initialized`, `PLAYER_ADDED`,
   `player_queues.on_player_register` (queue restored once per player lifetime, committed only if
   the player is still live). The pending cross-store cleanup list is drained here (§6i), with the
   cache available. A record whose stored provider has no manifest publishes a "provider not
   installed" `setup_reason`; `device_info.manufacturer` is filled from the stored device info, else
   the stored provider's manifest name, else its domain (HA dev `entity.py:30-35` calls
   `provider.name` on a possibly unloaded provider).
3. Webserver (`mass.py:347-350`): clients and HA receive one complete list, unavailable players
   included.
4. `_load_providers` (`389`): providers attach, one `PLAYER_UPDATED` each; derived bridge protocols
   are built by the bridge managers from the player's protocol records when their base attaches.
5. `mass.stop`: unload = detach all (events suppressed while closing); queues flush in
   `player_queues.close()` (`player_queues/controller.py:280-289`).

| Situation | Events |
|---|---|
| New device minted; boot of a stored player; enable | `PLAYER_ADDED`, `QUEUE_ADDED` (enable also `PLAYER_CONFIG_UPDATED`) |
| Attach, refresh, replacement, detach, provider reload, role flip | `PLAYER_UPDATED` |
| Disable | `PLAYER_CONFIG_UPDATED`, then `PLAYER_REMOVED`; the object goes, the record stays, HA drops the device (release note) |
| User removal; provider removal leaving no record; merge (absorbed id); session-only detach | `PLAYER_REMOVED` |
| Feature or control change | `PLAYER_UPDATED` plus the HA workaround `PLAYER_CONFIG_UPDATED` (`controller.py:2275-2288`) until HA re-evaluates features on `PLAYER_UPDATED` (HA dev `media_player.py:254-260`) |
| Command to an unavailable player | `PlayerUnavailableError`; power on accepted; play with an external power control powers on (`controllers/players/helpers.py:142-161`) |

## 4. Models and API

One models release 1.1.237, appended with explicit defaults: `legacy_ids: list[str] =
field(default_factory=list)`, `last_seen: float | None = None`, `unavailable_since: float | None =
None`, `protocols: list[PlayerProtocolInfo] = field(default_factory=list)`, new `PlayerProtocolInfo`
and `ProtocolRole` (with an unknown fallback), `IdentifierType.ZONE` (absent in 1.1.236, which ends at
`AIRPLAY_ID`). `forbid_extra_keys` is not set in 1.1.236's `player.py`. HA `dev` pins
`music-assistant-client==1.8.0` (`manifest.json:15`), which pins models 1.1.236 (client
`pyproject.toml:10`): the same version as the server (`pyproject.toml:29`,
`requirements_all.txt:57`), so additive fields are safe for HA. Server: pin bump in PR 2; the
exposure of `legacy_ids` on the player state and the `API_SCHEMA_VERSION` bump (`constants.py`) follow
the pin as a small PR of their own, with the first field the server sends to clients (a pin bump on
its own is no reason for it); the schema number is the next free one at that time, since dev raises
it independently. Legacy id resolution (PR 3): one `run_api_command(handler, args)` in `helpers/api.py`
(home of `APICommandHandler`, line 69) used by
all four dispatchers, `websocket_client.py:360-361`, `webserver/controller.py:889-890`,
`fastmcp_server/execution.py:888`, `snapcast/socket_server.py:190`; it rewrites `player_id`,
`queue_id`, `target_player`, `player_ids`, `child_player_ids` (`controller.py:1520`),
`player_ids_to_add` / `player_ids_to_remove` (`1447-1449`), `members` (`1606`), `source_queue_id` /
`target_queue_id` (`player_queues/controller.py:1301-1302`, before the permission checks at
1306-1307), and the player-filter arrays of `auth/user/create` and `auth/user/update`
(`auth.py:1358, 1693`); never protocol ids (setup flows name the protocol child,
`config/flows.py:640-677`). `get_player` (`controller.py:483`) and `player_queues.get` /
`queue_data` (`382, 387`) resolve for internal callers; `has_player_access` checks the id of the
looked-up player when the caller passes one, and the community MCP server's `player://` and
`queue://` resources resolve their target. New commands: `players/merge` (PR 5),
`players/remove_unavailable` (PR 4). `config/players/*` unchanged on the wire.

## 5. PR sequence

Outside the branch, now: (a) a dev PR removing the ten "remove after 2.10" steps (`migrations.py`
532, 539, 555, 583, 587, 601, 613, 625, 758, 1053); (b) models 1.1.237; (c) the shutdown flush bug
stays an independent dev fix. The local-audio step (`migrations.py:607`, `1307`, "remove after 2.11")
is not touched by this sequence: it runs before the split step in the same `migrate()`, so the id it
settles on is the one that gets frozen.

### PR 1. Pure state derivation
Landed as server#7508. Scope: new derivation module (`models/player_state.py`); `update_state`
(`models/player.py:2184`), `__collect_input_snapshot` (`2669`), `__calculate_player_state` (`2778`)
and the `__final_*` methods become `derive_player_state` with the input and effect boundary of §1.
The input snapshot reads `device_info`, `extra_attributes` and `extra_data` through the properties,
and the reported `sound_mode_list` and `options` lists themselves: no player overrides those two
properties, and each property read builds a new list on the hot path. `mark_external_source_ended`
(`2257`) keeps clearing the reported values itself until PR 4: providers read the raw values (Sonos
`source_list` and `play()`, Raumfeld's external-source tracking, the AirPlay control player, the
linking and announcement code), so a masking override in the derivation is not behaviour-neutral.
Today's rules only, plus the one-update lag of `group_volume` / `group_volume_muted`. No early XPASS:
`test_player_stays_available_on_its_audio_protocols` keeps failing because `available` still reads the
native's own value. Budget: a rewrite of the inputs, not a move (about 40 `get_player` calls between
2544 and 3759 become snapshot lookups). Tests: one per rule in `tests/models/`; the `_Player__*` pokes
in 9 files move to the function. No data, models, client change; no moves. Markers: none.

### PR 2. Identifier matching as one table, models pin
Scope: pin bump; `controllers/players/matching.py`: identifiers as sets of (type,
value) (a Raumfeld room reports several UUIDs, `tests/scenarios/test_identity.py:239`); veto
rules: a differing AIRPLAY_ID, CAST_UUID or ZONE vetoes across domains, a differing UUID or SERIAL
only when both come from the same domain (Chromecast reports UUID = cast uuid
`chromecast/player.py:154-155`, DLNA the UDN `dlna/player.py:461-463`, MusicCast and Bose their
`device_id` `musiccast/player.py:212-213`, `bose_soundtouch/player.py:141`; these pairs match by MAC
today, `protocol_linking.py:1942-1964`); "differing" means disjoint non-empty sets, so a shared claimed
identifier never vetoes; then MAC with the locally-administered normalisation, SERIAL, UUID, CAST_UUID,
AIRPLAY_ID; IP last under today's conditions restated in roles (`2024-2034` reads the universal
literal). `_identifiers_match` (`1903-2040`) and callers use it; `Player.identifiers` built from
`device_info`, `extra_data["reported_mac"]` and `cached_arp_mac`; the Sonos `_MR` suffix normalised
in `dlna/player.py`; wire `DeviceInfo.identifiers` unchanged. The sibling match (`333`) stays until
PR 4. Tests: `tests/controllers/players/test_matching.py`, one per rule and per issue (#6807, #5985,
#6847). Markers removed (run with `--runxfail` to confirm): `test_zones_of_one_chassis_are_one_player_each`
(needs only ZONE and the veto; natives never match each other today) and
`test_receivers_on_one_host_stay_separate_players[uuid]` (two DLNA renderers, same-domain veto, the
universal flow keeps them apart like the passing AirPlay case). No moves.

### PR 3. Legacy ids and the API id resolver
Landed as server#7504. Scope: §4: the reverse index and `resolve_player_id`, `run_api_command` in
the four dispatchers with the full argument list, internal lookups. The exposure of `legacy_ids` on
`PlayerState` (from `values.legacy_ids`, the other new fields at their defaults until PR 4 fills them)
and the schema bump follow PR 2's pin as a small PR of their own (§4). Tests: dispatcher tests through
both transports incl. permission handling of `player_queues/transfer`, state serialisation. Stored
data: `legacy_ids` written only by merges (PR 5). Can run in parallel with PR 1 and PR 2. Markers:
none. No moves.

### PR 3b. Permanent layers before the cutover
Everything below is end-state code whose body does not change in PR 4 except for a storage key.
(1) Scoped config access: `get_player_value` / `get_protocol_value` and setters, backed by the
single tree for now (`players/{id}` for both, one line per accessor changes in PR 4); complete
call-site conversion of the 73 getter and 25 setter calls (sendspin 8, airplay 7, cast bridge 4,
auracast 2, raumfeld 1, controller 3), the setup-data writers (`models/player.py:1258, 2513-2524`,
`config/flows.py:574-589, 672-677`), the direct readers `sendspin/bridge_manager.py:182, 279-302`,
`sendspin/provider.py:1865, 1983-1986`, `airplay/provider.py:1226, 1259`, `dlna/provider.py:116-117`,
`dlna/player.py:139-140`, `chromecast/sendspin_bridge.py:863-868, 1262-1264`; save routing by
declaring entry (a native whose id differs from its player's is written to its own record, which
today's unprefixed native entries at `config/players.py:1143-1152` plus `save_player_config` would
put on the player record). (2) Removal is a controller concern: `remove()` (`controller.py:1968-1997`)
unregisters permanently, calls the optional `on_remove()` hook, deletes the config; the six
`remove_player` overrides (`hass_players/provider.py:140-148`, `fully_kiosk/provider.py:103-120`,
`mpd/provider.py:90-107`, `msx_bridge/provider.py:207-214`, `snapcast/provider.py:655`,
`universal_player/provider.py:285-300`) and the two `remove_group_player` overrides
(`sync_group/provider.py:77`, `universal_group/provider.py:61`) become hooks; `REMOVE_PLAYER` on every
player provider (7 of 30 manifests declare it today; `REMOVE_GROUP_PLAYER` stays on the two group
providers, which the frontend checks for groups at `player_settings_actions.ts:213-214`). (3) The
remaining foreign-object lookups move to `mass.players.get_protocol` (`controller.py:507`):
`airplay/player.py:963-965, 972-976, 1068-1069`, `airplay/announce.py:401-404, 425-428`,
`airplay/provider.py:1078-1081`, `chromecast/sendspin_bridge.py:1000-1002`, `sync_group/player.py`
(17 sites incl. `_translate_to_parent_ids` 1103), `squeezelite/provider.py`. (4) `session_only`
reported by Sendspin (`is_web_player` `sendspin/player.py:2143`, `is_virtual_player`
`provider.py:788`, DISPLAY / VISUALIZER / LIGHT / SOURCE at `provider.py:1337-1344`,
`player.py:2903-2908, 3025-3036`, `sendspin_lights/base.py:81`, `hue_entertainment/bridge.py:126`),
no behaviour yet. Markers: none. No moves.

### PR 4. The cutover: Player and PlayerProtocol, boot from config, config tree split, universal provider removed
One PR, commits in this order, each reviewable on its own; the suites are green at the PR head
(commit 1 breaks the providers and tests that commits 6-9 convert): (1) model split (`Player` core half, provider half into
`models/player_protocol.py` replacing the alias, `__eq__`/`__hash__` per class, `protocol_id`
rename); (2) controller lifecycle: `_protocols` beside `_players` (`212`), attach / detach /
remove_protocol / remove replacing `register` (`1715`), `register_or_update` (`1840`), `unregister`
(`1905`); the guards of §6b; `iter_players(provider_filter)` (`391`) on the primary provider,
`__iter__` (`2741`) over players, `iter_protocols()`, `players/all(return_protocol_players)`
(`396`) and its callers (`msx_bridge/provider.py:140-144`, `models/player_provider.py:115-126`,
`models/player.py:838, 3768`, `diagnostics/__init__.py:232-237`) move to protocol iteration;
`_poll_players` (`3555`) polls protocols; `mass.unload_provider` (`mass.py:1189-1197`) detaches;
command routing, volume and mute ownership (§6c); commands to an unavailable player raise, power on
accepted (`cmd_power` 983-992), play with an external power control powers on; (3) config: the
tree split migration and the accessor switch (§6i), boot step and `get_player_configs`
(`config/players.py:161-208`) listing offline players with a reason, `get_player_config_entries`
(`282-290`) rendering a player without a live native, `config/players/get` and `save` for a live
session-only player without a record (first save promotes it: player record plus protocol records,
`session_only: true`), in-memory config for a protocol without a record (§6e), enable / disable /
remove as controller concerns (`on_player_config_change` 2586-2671: disable = detach, stop,
`PLAYER_REMOVED`, no object; enable = create from record (a session-only one is created by its next
attach), `discover_players()` of every protocol
record's provider; the cascade at 2617-2640 and the defaults `models/player_provider.py:66-77`
deleted; Sendspin, MSX and MusicCast re-offer connected clients from `discover_players` instead of
their `on_player_enabled` overrides at `sendspin/provider.py:563-574`, `msx_bridge/provider.py:190-205`,
`musiccast/provider.py:83`), the removal sweep (`config/providers.py:511-522`) on protocol records,
removing a player only when none remain, `last_seen` debounced on availability transitions,
`players/remove_unavailable`, manufacturer never empty; (4) linking: attach matches against players
(no universal branch), the collect window of §1, re-identification, derived protocols from
`underlying_protocol_id` and the records; (5) deletions: the universal provider
(`providers/universal_player/`, 766 lines, imports at `protocol_linking.py:51-52`, 84 literals), `ProtocolBackedPlayer` (`models/protocol_backed_player.py`, 215 lines), the universal
block `protocol_linking.py:573-1381`, the two-sided bookkeeping (`201, 333, 689, 720, 1038, 1528,
1555, 1634`), `_cleanup_player_type_transition` (`1829`), `_finish_player_type_transition`
(`controller.py:2882`), `set_player_type` (`config/players.py:595`) and the type write
(`models/player.py:2239-2240`), `_registration_aborted` (`2865`) and callers, the swap logic
(`1848-1881`), `_get_online_player`'s checks (`4690-4721`), the cached `linked_protocol_ids` branch
of `output_protocols` (`1883-1910`), `is_native_player`'s literal (`1811-1815`), `_take_over_mac_player`
(`sendspin/provider.py:1999-2031`) with its delete at 1750-1753, `carry_over_player_config`
(`controller.py:2054-2107`), `replacement_player_id` on `delete_player_config` (`1999`),
`_repair_protocol_parent_links` (`3469`), `_fix_group_member_configs` (`3503`) and its weekly task
(`342-352`), `_migrate_orphaned_disabled_protocol_configs` (`migrations.py:1742`), the
self-referential and ghost-player steps (`1427, 1522`), `CONF_LINKED_PROTOCOL_IDS`,
`CONF_PROTOCOL_PARENT_ID` (`constants.py:202-205`), `hass_players` `native_player_macs`
(`hass_players/helpers.py:84-97`, replaced by the native attach rule), the 45 s cached-parent wait and
the universal-player restore (`universal_player/provider.py:74-91`); (6) providers, mechanical:
call sites, role reports, `protocol_id`, `self.state` per the §6c table (sendspin 17, roku 5,
musiccast 3, squeezelite 2, dlna 2, wiim, sonos, snapcast, chromecast, alexa, airplay 1 each);
(7) providers, per site: the `protocol_parent_id` readers (`sendspin_source/provider.py:524,
789-790`, `ariacast_receiver/__init__.py:1038-1040`, `airplay/sendspin_bridge.py:1768-1770`,
`chromecast/sendspin_bridge.py` 5 sites incl. 1000-1002 and 1160-1164, `sendspin/player.py:2294-2296`,
`sendspin/provider.py:1575, 1857`, `sync_group/player.py`, `chromecast/player.py`) move to
`protocol.player`; the `PLAYER_REMOVED` subscribers keep their meaning (removal only):
`spotify_connect/provider.py:624-635` and `airplay_receiver/provider.py:398-409` already keep daemons
across `PLAYER_UPDATED`, `plex_connect/__init__.py:552`, `ai_radio/queue_dj.py:264, 278`,
`yandex_smarthome/notifier.py:155-157`, `sendspin/player.py:2964`; the Cast bridge
`_on_player_unregistered` (`chromecast/sendspin_bridge.py:1160-1164`) re-evaluates on availability
transitions, since the parent now always exists; LinkPlay (§6c); group-member and leader lookups in
Sonos, Squeezelite, Snapcast, MusicCast, AirPlay, WiiM, sync_group; (8) test infrastructure: the
harness (`_ROLE_TYPES` 121-124, `attach` 491-503, `detach` 505-514, `discover_players` 354-360,
`remove_player` 362-364, `player_of` 771-786 via `protocol.player`, `protocol_config` 827-835 reads
`player_protocols/`, `_carried_ids` 1310-1312, `protocol_id`), `tests/common.py:411` `MockPlayer(Player)`
becomes `MockProtocol(PlayerProtocol)` plus an `attach_mock()` builder; (9) test rewrites;
(10) scenario markers and rewrites.

Deleted scenario behaviour rewritten: `test_airplay_and_cast_of_one_device_become_one_player`
(player appears at the first attach because both carry a MAC, the second protocol joins;
`provider` the airplay instance), `test_detached_native_player_vanishes_also_after_restart`
(becomes: stays listed unavailable, back after restart, unavailable until the provider attaches),
`test_harness.py::test_idle_fails_while_a_cached_parent_is_awaited` (no 45 s wait),
`test_player_of_follows_the_attachment` (`player_of(den_airplay)` is a minted player, not None).
Markers removed: 20 cases (§7). Stored data and migration: §6i. Models/API: `protocols`,
`last_seen`, `unavailable_since` filled. Frontend: landing dependency "gate on player state"
(§6, frontend note below). HA: entities go unavailable instead of vanishing; disable removes the
device. Tests: `test_protocol_linking.py` (12,814 lines, 426 `MockPlayer(`, class at 166): universal,
takeover, merge, cached-parent and bookkeeping tests deleted, the rest on the builder;
`test_player_controller.py` (8,749 lines): `TestUnregisterCleanup` 1149, `TestRegisterUnregisterRace`
1250, `TestRegisterFailureRollback` 1516, `TestRegisterOrUpdateTypeTransition` 1717 rewritten per
§6b, `TestPlayerAvailability` 539, `TestPlayerCommandAvailability` 669, `TestOfflinePowerControl`
3800, `TestUnregisterTeardown` 8332, `TestDeletePlayerConfigUserFilters` 8351,
`TestDeletePlayerConfigGroupMemberships` 8379 adapted; 24 files on `_players` (8 via
`players._players`), 9 on `_Player__*`, 14 on `universal_player`, 31 on `protocol_parent_id`, 15 with
`"players/` literals, 4 on `linked_protocol_ids`; `tests/providers/universal_player/` and
`test_carry_over_player_config.py` deleted; `test_register_creates_config.py` (64-87) rewritten for
attach; migration fixtures (§6i). Hardware: Squeezelite on a smart plug (#7435 end to end); HomePod
mDNS flap; Sendspin same-object hello, replacement connection, ESPHome MAC → key id while customised;
a Sendspin-only ESP32 across the upgrade with a real settings.json, two restarts; a browser player
(static delay reported, nothing stored); Cast TV with DLNA and MusicCast with DLNA still one player;
a Pi with Squeezelite and a Snapcast client stays two players; a sync group after the migration;
disable and enable a Sendspin and an MSX player; enable a player disabled before the upgrade;
Chromecast provider reload with HA connected; a new Sonos: change a setting, restart, applied;
AirPlay pairing and output settings across migration, restart and provider removal; Sonos native
drop with AirPlay up (#6951) incl. volume; Snapcast host with Sendspin playing (volume routing,
#6468); LinkPlay native grouping plus DLNA playback; WiiM provider removed while AirPlay attached;
output switching while muted and at the volume limits; restart with every provider disabled, enable
one, remove one from the frontend with HA connected; boot time and HA entity count on a large
settings.json. No moves (the provider half moves into an existing file; new modules only).

Carried over from the reviews of PR 1 to PR 3. `mark_external_source_ended` converts together with
the providers that read the raw values it clears (PR 1); proposed: a hook on the protocol that the
core calls, whose default clears the three reported values. The input collector keeps PR 1's shape
when its inputs come from protocol objects (§6h). Protocols are looked up in `_protocols`, never
through the legacy-id resolver: `get_protocol` goes through `get_player` today, and a frozen id is
shared by a player record and its protocol record, so after a merge the resolver would hand out the
keeper for a protocol id. The resolver's guard (an id with a stored config is never a legacy id)
checks player records only, since an absorbed frozen id keeps its protocol record and must still
resolve to the keeper. Tests: a protocol whose id is another player's legacy id; after a frozen-id
merge, the absorbed id resolves to the keeper as a player and queue id while its protocol is still
found by that id. The vetoes apply to the
merged identity of a player (§6f): the sibling match and the seed-only collection in the linking
code compare pairwise and can ignore a veto, and both are replaced; identifiers are kept per
protocol record, so "the same domain" is exact where PR 2's matcher approximates it as "the two sides
share any domain" (a wrapper stores one value per identifier type). The grouping and redirect code
carries over the access rules for grouped players (plan, Lifecycle).

### PR 5. Auto-merge
Scope: `merge_players(keep, absorb)` and `players/merge`; the automatic merge at attach when the
younger player is unused: no custom config (no `name`, no user-editable key on its player record or
its protocol records, no `player_dsp/` record with DSP enabled, no `player_queues/` record; the
automatic keys of §6e never count), no saved queue (a `CACHE_CATEGORY_PLAYER_QUEUE_ITEMS` row with
at least one item: items are written only when `items_cache_dirty` is set by an items change,
`player_queues/controller.py:1894, 2095`, whereas the state row is written for every queue at the
first flush, `2086`, `state.py:120`; a row that expired after 30 days without a write,
`cache/constants.py:12`, counts as unused, accepted), no group membership (not in any group
protocol's member lists, not synced, not in an active group); age = `created_at`, a record without one
is older, a frozen id wins a tie; the shared-domain veto stays. Legacy ids have one writer, the
merge: the keeper gets the absorbed player's id and the absorbed player's own legacy ids,
de-duplicated and persisted, and the merge rebuilds the reverse index itself before the absorbed
config is deleted (otherwise it is rebuilt at setup and when a player config is deleted). A queue
command called inside the server with a legacy id (a provider setting that still names an absorbed
player, such as the Party plugin's player or a Yandex Station target) indexes the queue store with
the id it was given, so the public queue commands work on the queue's own id from their first line
on, as the read accessors already do. Stored data: `legacy_ids` appended; no migration. Markers
removed: 3 cases (§7). Tests: merge policy per signal; a chain (A merged into B, then B into C) keeps
A resolving; the two legacy-id queue commands above; the scenario rewrite of §7 for the used pair.
Frontend: "merge into…" follow-up. Hardware: TV seen as AirPlay first with a late
address, then Cast. No moves.

### PR 8. Zones, claimed instances, native priority
Scope: providers report ZONE (Bluesound zones keyed by host and port, `bluesound/player.py:264`,
identifiers at 76-80; MusicCast zones `musiccast/player.py:190-192`; AmpliPi); a native protocol claims further instances of one domain,
attached hidden, one output row (Raumfeld room reports its renderers' UDNs,
`raumfeld/provider.py:269-279`); a second native attaches as control with the priority vendor native
> hass_players > generic. Markers removed: `test_room_of_several_renderers_is_one_player`. Hardware:
Raumfeld room of three renderers, NAD or MusicCast zones, Sonos with its HA entity. No moves.

### PR 9 onwards. Structure (bucket B), file moves allowed from here
9a grouping as `plan_group(leader, members) -> GroupPlan` plus one executor, replacing the
`_try_group_via_*` chain (`protocol_linking.py:2905, 2952, 3004` and the translation from 2317) and
the controller's `set_members` paths; 9b the 30 s lock bypass (`controller.py:2772-2784`) fails
loudly, timers as explicit per-player state; 9c controller split by concern, `protocol_linking.py`
split into matching (PR 2's module), output selection, grouping translation; 9d the players README
as the contract and the unit suites moved onto the harness; 9e the `noqa PLR09xx/C901` hotspots fall
out of 9a-9c. Model compatibility aliases: outside this sequence. Companion PRs: frontend "gate on
player state" (lands with PR 4; `player_settings_actions.ts:68-78, 140, 209-219`, `Players.vue:162-167`
gate on the stored `PlayerConfig` instead of `api.getProvider(...)`, live state from
`api.players[playerId]` only where present (a disabled player has none); without it the settings
editor and enable/disable are hidden for a player whose provider is unloaded or uninstalled, and
delete needs a loaded provider advertising the feature; the settings list already works, since
`player_config.ts:14-15` asks `include_unavailable=false`, which after PR 4 only hides session-only
players); later "merge into…", "remove all unavailable players", and dropping `REMOVE_PLAYER` from
every provider once the frontend stops asking. HA later: #418 id move on `legacy_ids`; features
re-evaluated on `PLAYER_UPDATED`, then the server drops the workaround at `controller.py:2275-2288`.

## 6. Resolutions of the review questions

**(a) Ordering.** Recommendation: the config tree split, the boot step and the session-only step
move into PR 4 (v1's PR 6 and PR 7 become its commit 3), re-identification too; auto-merge stays
PR 5. Opus's 4 → 6 → 7 → 5 keeps a single tree for two PRs, so PR 4 and PR 6 would carry a boot
classifier (edge → protocol record, no edge → player record that is also its same-id native), a
shared-record branch in the removal sweep, the enable cascade and the separate-id native's config
routing, all rewritten by PR 7: throwaway work the rules forbid. The classifier is also unreliable
on real data: 11 of the 23 native records locally hold only automatic keys, indistinguishable from a
controller-only record. With split records PR 4 deletes `_take_over_mac_player` and
`carry_over_player_config` outright, since re-identification is a record takeover plus an edge; the
local-audio step is not an id change after the freeze because it runs before the split step in the
same `migrate()`.

**(b) Race guards.** Recommendation: one attach token per protocol slot plus the kept teardown
checks; the queue commit becomes conditional. Mapping of today's guards: the post-lock and
post-await teardown checks (`controller.py:1723, 1764`) stay as they are, since a provider can unload
without any slot changing; `_registration_aborted` (`1785, 1793, 1831`) becomes `self._attaching[id]
is token` after every await, where detach and replacement of that id replace the token (per slot, so
detaching protocol B never aborts A's attach); the rollback (`1798-1813`) unbinds the slot if still
ours, runs `on_detach` best effort and re-raises, and removes a player minted in this attach that
has no record yet; the "not the registered instance" check (`2174`) moves into
`PlayerProtocol.update_state()`, which forwards only while `player.protocols[protocol_id] is self`
(`on_detach` clears `player`); queue restoration runs once per player lifetime at boot or mint and
`on_player_register` (`player_queues/controller.py:1427-1469`) commits `_queue_data` only if the
player is still live, else drops it; `_register_lock` (`227`) stays, so a replacement waits for an
in-flight attach. Every test of `TestRegisterUnregisterRace` (1272-1448) and
`TestRegisterFailureRollback` (1528-1694) maps to one of these and is rewritten, none dropped.

**(c) Provider conversion contract and volume.** Recommendation: a four-line contract in
`DEVELOPMENT.md` and a per-file checklist in the PR. A protocol reads its own reported values through
its own properties, never a published state; it reads `self.player.state` only for what the device
must display or follow (current media and position for the metadata pushes at `airplay/player.py:1042`,
`chromecast/player.py:389`, `sendspin/player.py:1582-1588`), never for its own reachability
(`snapcast/player.py:655-659` compares its previous reported value instead); topology (`group_members`,
`synced_to`, `active_group`) comes from `self.player.state`; transient data stays on the player.
`ProtocolBackedPlayer` is not replaced by forwarding properties: its aggregation (`48-130`) is the
derivation, its external-source transport commands (`131-170`) are the controller's
`_resolve_command_target` / `_forward_to_external_source` (`controller.py:5088-5159`). LinkPlay
(`wiim/linkplay_player.py:44`) becomes a control-role protocol reporting power, volume, mute, native
availability (`138`) and grouping, with a reported native-follower flag the derivation consumes;
its delegated playback properties (`173-196`) and `_backing_protocol_player_ids` (`301-303`) go.
Volume and mute: the player owns the logical level and mute (`PlayerState.volume_level`), scaling
stays in the controller (`2109, 2117`); the control target per the plan rule (active output, else
native, else any available: a change from `_get_protocol_player_for_feature`, `models/player.py:2552-2622`,
which prefers native today, the #6468 case); a protocol receives the logical level and mute on
activation before the stream starts; feedback is accepted only from the current control target.

**(d) Availability and detach.** Recommendation: two predicates, as today. `state.available` =
enabled and (any protocol available or `offline_by_power_control`) and not needs_setup
(`models/player.py:2816-2818` with "any protocol"); `available_for_playback` = a reachable, set-up
protocol exists (`644-656`); `offline_by_power_control` (`660-674`) unchanged except the type check.
Session-only detach always removes the live object and keeps a customised record
(`tests/scenarios/test_lifecycle.py:199-225`). Queue quiescing happens only when the detached protocol
was the playback path (the active output, or the native rendering): today's `on_player_remove`
(`player_queues/controller.py:1553-1573`) without the `_queue_data` pop and without purging, the
queue goes idle with its resume position; the detach of an idle control or native protocol while
another output plays touches nothing. Group release stays on the availability transition
(`controller.py:3257-3271`); announcement restoration keys on the player, which now survives.

**(e) Keys and transient config.** Recommendation: classify by who declares the entry. Core default
entries (`config/players.py:776`) and the player-owned metadata are player keys; every entry a
protocol declares, the per-output audio entries (`output_channels`, `output_codec`,
`prefer_wav_for_live_sources`, `http_profile`, `enable_icy_metadata`, `sample_rates`, `flow_mode`,
`flow_mode_sample_rate` as rendered at `716-775`) and provider-written keys are protocol keys. The
sync-group member lists therefore live on the sync-group protocol's record and the membership
rewrites on remove and merge (`controller.py:2023`) read group protocol records. `cached_arp_mac` and
`reported_mac` stay per protocol (`_resolve_mac_addresses`, `2805-2855`, runs per protocol at attach
and matching an unbound protocol needs them). A protocol without a record (session-only, not
customised) holds an in-memory `PlayerConfig` (`set_config`, `models/player.py:2337`); provider writes
such as `sendspin_static_delay` (`sendspin/player.py:1556, 1765`) update it without persisting, the
MAC cache is skipped, and only a user save promotes it, so neither `set_raw_player_config_value`'s
`KeyError` (`config/players.py:662-665`) nor a nested `config.set` creates a partial record.

**(f) Natives and vetoes.** Recommendation: a native attaches only to a player without a native
protocol record, or takes over the player's offline native record of the same provider and domain
(re-identification, §1); a player holding a native of another domain stays separate until PR 8's priority
rule (a Pi with Squeezelite and a Snapcast client, a hass_players entity of a native device). Vetoes:
UUID and SERIAL same-domain only, AIRPLAY_ID, CAST_UUID and ZONE cross-domain, all over sets.
A veto applies to the merged identity of a player, not to one pair of protocols. "The same domain"
means both values were reported by a protocol of that domain.

**(g) Models and the resolver.** Recommendation: the pin in PR 2 (it needs ZONE), the resolver in
PR 3, the fields and the schema bump in a small PR after the pin. HA `dev` already runs models
1.1.236 through client 1.8.0, so the v1 note about a 1.1.189 pin is stale. PR 3's inventory is §4,
incl. `player_queues/transfer`, the user-filter arrays,
the fastmcp and snapcast dispatchers; protocol ids are never rewritten.

**(h) PR 1's boundary.** Recommendation: §1's input list and effect list, today's rules only. The
derivation takes the previous state, the anchors and the clock as inputs and returns effects; timers,
persistence and callbacks stay in `update_state`. `test_player_stays_available_on_its_audio_protocols`
stays a strict xfail through PR 1 because `available` still reads the native's own value. The
collector resolves artwork only for the queue the media is taken from (`resolve_media_queue`) and
collects a value only for the role that reads it (`GroupSnapshot` for groups,
`available_for_playback` only for members); PR 4 keeps that shape when the inputs come from protocol
objects.

**(i) Migrations.** Recommendation: one run-once flag per step (`player_split_session_only`,
`player_split_tree`), written into the data by the step and saved with the rest of `migrate()`'s
result (`config/controller.py:323-324`); backup as a prerequisite. Backup: in `_load`, before
`migrate()`, when the tree flag is absent and no backup exists yet, copy the loaded file (it can be
`.backup`, `config/controller.py:318`) to `settings.json.pre-player-split` through a temporary file
and a rename, so a retry keeps the first backup and a crash leaves no partial one; if the
copy fails the split steps do not run and startup stops with a clear error (continuing would read
the new tree from unmigrated data and show no players; see Q1). Session-only step (old shape only):
candidates are records with provider domain `sendspin`, `player_type` other than `protocol`, no
`protocol_parent_id`, no record pointing at them; that is web and app players, virtual players,
displays, visualizers, lights and sources, because every hardware Sendspin client is protocol-typed
today (`sendspin/player.py:2168-2176`). Locally: 137 such records, 0 names, 81 empty, 56 with only
`sendspin_static_delay`, 2 with a `player_queues/` record, 5 lights. Customised = stored `name`, any
user-editable key (player-level list or declared protocol entry) in `values`, a `player_dsp/` record
with DSP enabled, or a `player_queues/` record; automatic keys never count (`cached_arp_mac`,
`reported_mac`, `linked_protocol_ids`, `protocol_parent_id`, `created_at`, `device_identifiers`,
`device_info`, `last_seen`, `legacy_ids`, `session_only`, and the provider-written keys
`sendspin_static_delay`, `sendspin_opt_out_pending`, `sendspin_unsupported`, `stored_volume`,
`source_input_declined`, `protocol_experimental_note`, `underlying_player_id`). Delete uncustomised
ones with their DSP and queue records and append their ids to a top-level
`player_split_pending_cleanup` list; mark customised ones `session_only: true`. The boot step drains
the list: saved queues of both cache categories (`purge_saved_queue`, `player_queues/controller.py:1576`)
and user filters (`remove_from_user_filters`, `auth.py:1607`; `_prune_stale_user_filters` at 2485 is
a no-op without players and maps no ids), then clears it; a crash in between re-runs it. Tree step:
restore a missing child edge from the parent's `linked_protocol_ids` when the child record exists and
no other parent lists it, a wrapper's list outranking a native's (music-assistant/server#7156; all 7
empty `up…` wrappers locally are this half-write); then `up…`
records (`provider == "universal_player"`, never by type) get `provider` = highest-priority child
record, wrappers still without a child are deleted with DSP and queue records and added to the
cleanup list; protocol-typed records → `player_protocols/{id}` with `player_id` = edge, `role` =
audio, one without an edge unbound (attach re-matches) with `enabled: true`, since its flag was the
device's own enable; every other player record's provider-owned keys, audio keys and `setup_data` → a same-id
`player_protocols/{id}` with `role` = native (virtual for the sync_group, universal_group and
auracast domains); a former `up…` wrapper (stored provider `universal_player` when the step starts)
gets none, since it has no protocol of its own and a native must still be able to attach to it
(§6f); `linked_protocol_ids` and `protocol_parent_id` dropped. Repairs in the pass:
a stored UUID identifier ending in `_MR` loses the suffix (the UDN of a Sonos renderer, which the
DLNA provider reports without it since PR 2; a record without a MAC and with an old IP otherwise
no longer matches its Sonos player);
edge to a missing, self or child record → null (unbound, attach re-matches); two protocols of one
domain on one player → keep the enabled one, else the one with `setup_data`, else the lowest id, null
the rest (one local parent has three enabled AirPlay children without setup data); a record with an
edge but `player_type: player` is a protocol record (`config/players.py:632` never updates the type);
group member lists holding protocol ids → the edge's player; protocol records of disabled players get
`enabled: true` (the cascade at `controller.py:2630-2640` is gone; a per-output toggle under a
disabled player is lost, stated in the PR). Precedence for a partial split: a key present in both
trees keeps the new tree's value, old-only keys move, then the old record goes, so a crash between
the two writes converges. Edge validation against the strong-identifier rules happens at attach,
where the identifiers are: a stored edge whose player's attached protocols veto the newcomer is
dropped and re-matched. Fixtures: `settings.json` of #6807, #6847, #6461, #6737 and the local shapes
(uncustomised, customised, light, non-dict record, empty wrapper, half-written wrapper, dup-domain
parent), each run twice. Diagnostics dump: player → protocols with role and availability.

**(j) Marker ledger.** §7. The used-duplicate pair must not pass by the mere absence of merging:
after PR 4 both players stay by construction, so the strict markers would XPASS. Recommendation: on
the scenario branch, before PR 4 lands, `test_used_duplicate_is_not_merged[played,renamed]` gets an
unused control pair in the same scenario (a second TV whose younger player is unused and must merge,
its id resolving to the older one), so the case fails until PR 5 implements the policy and passes only
when the policy distinguishes.

## 7. Scenario markers per PR (26 cases)

| PR | Cases |
|---|---|
| 2 | `test_zones_of_one_chassis_are_one_player_each`; `test_receivers_on_one_host_stay_separate_players[uuid]` (both confirmed with `--runxfail` on the PR; a case that does not flip moves to PR 4) |
| 4 | phase 1 (15): `test_detached_native_player_stays_listed_unavailable`; `test_player_stays_available_on_its_audio_protocols`; `test_offline_device_is_powered_on_through_its_power_control`; `test_play_on_a_switched_off_player_powers_it_on`; `test_command_to_an_unavailable_player_raises[play,pause,stop,volume_set,volume_mute,play_media]`; `test_power_on_of_an_unavailable_player_is_accepted`; `test_player_outlives_its_removed_provider_on_its_other_protocols`; `test_disabled_by_default_device_is_no_player_until_enabled[airplay_mac,cast_tv]`; `test_session_only_web_player_exists_while_attached`. Phase 2 (5): `test_protocol_only_player_names_the_provider_of_its_protocol`; `test_reflashed_device_keeps_its_player`; `test_device_with_a_new_protocol_id_keeps_its_player_and_settings`; `test_role_flip_of_a_protocol_keeps_its_player[audio_to_native,native_to_audio]` |
| 5 | `test_unused_duplicate_is_merged_into_the_older_player`; `test_used_duplicate_is_not_merged[played,renamed]` (rewritten per §6j) |
| 8 | `test_room_of_several_renderers_is_one_player` |

Kept green through every PR: `test_customised_session_only_player_keeps_its_config`,
`test_pre_split_universal_player_config_comes_back_as_the_same_player[complete,half_written]`,
`test_protocol_only_player_is_restored_with_its_id_after_a_restart`,
`test_player_without_other_protocols_goes_with_its_removed_provider`,
`test_native_device_attaches_as_available_player`, the AirPlay veto case. Harness self-tests that
assume today's internals are rewritten in PR 4 (§5).

## 8. Decisions on the open questions

1. **Backup failure.** Startup stops with a clear log line and retries on the next boot; no
   migration without a backup.
2. **Outputs of players disabled before the upgrade.** The migration re-enables their protocol
   records.
3. **Enable of a push provider's device.** Enable calls `discover_players()` of each protocol
   record's provider; Sendspin, MSX and MusicCast re-offer connected clients there, and their
   `on_player_enabled` overrides go.
4. **Scenario rewrite for the used-duplicate pair.** Rewritten on the scenario branch before PR 4
   (§6j), so it cannot pass by the mere absence of merging.
5. **PR 3b.** Lands as its own PR before the cutover.
6. **A split step that fails.** A record that cannot be converted is logged and skipped. A step
   that fails as a whole stops startup like a failed backup; the next boot retries.
