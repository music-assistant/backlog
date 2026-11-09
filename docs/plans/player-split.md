# Player split: one player per device, providers attach endpoints

Status: decided with the maintainer on 2026-11-09. Epic: #412.

## Problem

A `Player` today is two things in one object: the player that users, Home Assistant and the queue
see (id, config, queue, controls, groups, output choice, derived state), and the provider's live
handle on a device. A player therefore only exists while a provider has it registered.

- A switched-off device disappears whenever its provider drops it: Squeezelite, Sendspin and
  AirPlay on disconnect, MSX after an idle timeout, every provider on a reload, and every native
  provider after a server restart. Power on through a smart plug (music-assistant/server#7435) cannot reach it, and Home
  Assistant cannot call it either (HA skips unavailable entities in service calls).
- Core state lives on the provider's object and dies with it: linked protocols, active output
  protocol, sleep timer, fake power/volume/mute. `register_or_update` loses the protocol links on a
  swap.
- Devices without a native provider need a stand-in identity, which is the universal player
  provider (710 lines) plus about 1,200 lines of linking, merge and takeover in
  `protocol_linking.py`. A native player taking over a universal player changes the id: HA gets a
  new entity, the saved queue is purged, Spotify Connect, AirPlay Receiver, Plex Connect, party and
  Yandex silently lose the device.
- Four providers unregister on device loss (Squeezelite, Sendspin, AirPlay, MSX), the rest stay
  registered and flag unavailable, so the same situation behaves differently per provider.

## Target model

**Core player** (`Player`, owned by the players controller, what `mass.players.get_player()`
returns). Identity, config, queue binding, power/volume/mute control resolution, output protocol
selection, group membership, sleep timer, transient flags, and the state derivation (today's
`@final` half of `models/player.py`, about 90 members). Persisted device data: name, device info,
identifiers (universal players already persist these). Created from config at startup, or by the
first endpoint of a new device. Removed only when the user removes it or disables it.

**Endpoint** (provider object, today's provider half: `_attr_*` state, commands, config entries,
setup flows). Roles, reported by the endpoint itself because one class can play several roles per
device (Chromecast, Squeezelite, Sendspin, AirPlay):
- native: the vendor's control plane, usually with its own audio path;
- native without audio: control and grouping only (WiiM/LinkPlay today);
- protocol: an audio path (AirPlay, Cast, DLNA, Sendspin, Squeezelite bridges);
- virtual: no device (sync group, universal group, auracast).

Every player is a core player, with no exceptions: native devices, protocol-only devices (today's
universal players), Cast groups and stereo pairs, sync groups and universal groups (virtual
endpoints). A core player has at most one native endpoint and at most one endpoint per protocol;
none is required.

An endpoint attaches to exactly one core player. One persisted edge records it: the endpoint's
`protocol_parent_id` (already the canonical side today), so endpoint configs stay where they are
(`players/{endpoint_id}`) and no data moves.

**Player id.** Owned by the core player, stable for its whole life, never derived again and never
switched by an endpoint. A one-time migration freezes the current visible id of every existing
player (RINCON_…, Cast uuid, `up…`) as its core id, so HA entities, user filters, groups, DSP,
queues and plugin bindings stay as they are. A player created after the change gets an
MA-minted opaque id. Moving existing players to MA-minted ids, with a clean HA migration, is a
later step.

**Lifecycle.**
- `register` / `register_or_update` = attach (or re-attach) an endpoint. A non-permanent
  `unregister` = detach: the core player stays, turns unavailable (or "available, powered off"
  under an external power control), keeps its queue. Permanent unregister = remove.
- Detaching releases the queue's audio and pending next-track work (today's `on_player_remove`
  part from music-assistant/server#7268) without dropping the queue.
- Plugins that use `PLAYER_ADDED` as "device back" or `get_player() is None` as "device gone"
  (Spotify Connect, AirPlay Receiver, Plex Connect, Yandex, Music Quiz, party, Cast Sendspin
  bridge) switch to the availability change that `PLAYER_UPDATED` already carries.
- Provider lookups of their own objects go through the provider (`self.get_endpoint(id)`,
  `self.endpoints`), never `mass.players.get_player()`. About 185 call sites, mechanical except
  group-member and leader lookups (Sonos, Squeezelite, Snapcast, MusicCast, AirPlay, WiiM,
  sync_group) that need a per-site look.

**What goes.** The universal player provider, the universal create/merge/takeover paths and the
two-sided link bookkeeping in `protocol_linking.py` (about 1,150-1,250 of 3,290 lines),
`ProtocolBackedPlayer` as a provider base (its behaviour moves into the core player), the 45 s
"wait for cached parent" reservation and `_get_online_player`'s "registered again" check.
What stays: identifier matching, derived (bridge)
protocols, output selection, control resolution and protocol grouping (about 1,500 lines).

## Decisions (2026-11-09)

1. **Delivery**: one release. Phase PRs target a long-lived `feature/player-split` branch off dev,
   which merges dev in regularly and lands on dev as a whole. No CI change: `test.yml` runs by
   hand (`workflow_dispatch` with `ref`) on the feature branch.
2. **Which configs come back at startup**: every enabled player config of a loaded provider,
   except session-only players (Sendspin browser/app and virtual players, displays, visualizers,
   lights, line-in sources), which keep "exists while attached". A one-time migration marks the
   existing Sendspin standalone configs; from then on the endpoint reports it. Stale devices show
   as unavailable until the user removes them.
3. **A native endpoint for a device that already has a core player** attaches to it; the id does
   not change. The universal takeover path goes.
4. **Naming**: the core object is `Player` (what `mass.players.get_player()` returns); providers
   subclass `PlayerEndpoint` and find their own objects through the provider
   (`get_endpoint(id)`, `endpoints`). The MSX Bridge upstream repo needs the same rename.
5. **Ids**: existing ids are frozen as core ids; new players get MA-minted ids (see above).

Defaults for the smaller questions:
- `PlayerState.provider` for a protocol-only core player stays `"universal_player"` until the
  frontend no longer needs it (the frontend's delete check and settings link read it).
- Two core players that turn out to be one device are still merged (older wins), as today.
- Last known device info and identifiers are stored lazily on the core player's config at point of
  use, no extra provider requests.
- Providers that can power on an offline device themselves (Sonos Wake-on-LAN) get an endpoint
  hook later; external power controls cover the rest.

## Phases (PRs into `feature/player-split`)

0. **Preparation, no behaviour change** (#413, #414; several small PRs):
   - provider-scoped endpoint lookup, and all providers moved to it (still returns today's object);
   - plugins move to availability transitions;
   - `Player.__init__` stops writing config (registration creates it);
   - core persists last known device info/identifiers.
1. **Core player + attach/detach for native endpoints** (#415). The registry holds core players, provider
   objects become `PlayerEndpoint`s; boot from config per decision 2; the one-time id migration;
   detach instead of unregister; queue lifetime follows the core player. Fixes Squeezelite on a
   plug and every native player after a restart.
2. **Protocol endpoints on core players, universal player provider removed** (#416). Universal configs
   become core players with the same `up…` ids (idempotent migration that never raises: identify
   them by `provider == "universal_player"`, never by type); takeover becomes attach.
3. **Cleanup** (#417). `ProtocolBackedPlayer` folded into the core, sync_group onto a core "endpoint for
   domain X" API, test suites rewritten around attach/detach (`test_protocol_linking.py` is 12.7k
   lines, 28 test files poke `_players`).
4. **Later, after the release** (#418): move existing players to MA-minted ids together with a migration
   in the Home Assistant integration (HA otherwise drops and recreates the entities).

A shutdown bug found while researching this (queue state of the last 5 s lost because providers unload
before the queue controller flushes) is independent and goes to dev as its own fix.

## Risks

- Size: 27 provider packages (15 S, 7 M, 4 L: AirPlay, Chromecast, Sendspin, sync_group), plus the
  linking tests. Phase 0 makes the later phases mostly core-only.
- Clients see more players: unavailable players stay listed during provider reloads and after a
  restart. Intended, but the frontend and HA will show it.
- Daemon-per-player plugins (Spotify Connect, AirPlay Receiver) would start for offline players at
  boot unless they check availability.
- Config migrations run every start before providers exist and abort startup on an exception:
  every new step needs a guard and must be shape-idempotent.

## Out of scope

Frontend and models changes beyond what compatibility needs, an auto-forget for stale players,
offline power hooks per provider.
