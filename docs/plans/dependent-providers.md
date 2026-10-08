# Dependent providers: technical plan

Status: proposal 2026-10-02. Owner: Marcel van der Veldt. This document is the technical companion
of the "Set up dependent providers in one go" epic (music-assistant/backlog#215) on the project
board. It follows the research in
music-assistant/backlog#201, which concluded that a provider keeps exactly one type. Every claim
marked "verified" was checked in the referenced code (server at `0693616`, models at `1.1.214`,
frontend at `main` on 2026-10-01).

## Summary

A service that offers several things is split into several providers that depend on each other
through `depends_on`: Home Assistant is a plugin plus a player provider, Plex is a music provider
plus a plugin, and there are six more pairs. The split is right: the halves have a different
cardinality (one Plex library, many Plex Connect players) or a different ownership regime (a member
owns a music source, a plugin is household-wide), and a dependent such as the Home Assistant
players provider is optional by design. What is wrong is how the user meets the split: adding a
dependent whose parent is missing aborts and sends the user back later, and the dependents of a
freshly set up provider are nowhere to be found.

This epic keeps one type per provider and `depends_on` as it is, and changes two things:

1. Setup flows chain. Adding a dependent sets up its parent first, directly when the parent needs
   no input or through the parent's flow, and then continues into the dependent in the same
   dialog. When a provider's setup finishes, the dialog offers its dependents. A dependent's row
   and its entry in the add dialog say which provider it needs.
2. The media methods that `PluginProvider` and `MetadataProvider` copy from `MusicProvider` move
   into shared capability mixins, so the overlap has one home and the controllers lose their
   union casts. Nothing becomes a music provider that is not a music source.

No server change for the chaining, no models change, no data migration, no change to the setup
flow protocol. Older clients keep working unchanged.

## Decisions taken with the maintainer (2026-10-02)

1. One type per provider; `ProviderType` and `depends_on` stay as they are (research #201).
2. No automatic creation of dependents, not through a manifest flag and not as a default provider
   on the add-on. The Home Assistant plugin is loaded by default on the add-on; its players
   provider is and stays optional. Whatever a user wants, the user adds, so the work goes into
   making that one path smooth.
3. Chaining is done in the frontend from the manifests and configs it already has. The server
   keeps the `missing_dependency` abort as the guard.
4. No "companions" line on the parent's card: the term is internal and the information is hard
   to phrase for users. The finish screen of the parent's setup carries the dependents with their
   own descriptions instead, and the dependent's card says "Needs <parent>".
5. The shared media methods become four mixins in `music_assistant/models/media_capabilities.py`
   (catalog, recommendations, music discovery, audio stream). No "helper music provider" is
   extracted (see Architecture, section 2).
6. Board: a standalone epic on the board. Sub-issues are plain issues in the backlog repo, not
   board items.

## Requirements

- Adding a provider whose parent is missing sets up the parent and then continues into the
  provider in the same dialog, in the settings and in the onboarding wizard. The user never has
  to come back.
- A parent that is configured but currently not loaded is not set up again.
- After a provider's setup finishes, the dialog offers the dependents the user may add.
- A member never sees or is offered a dependent they may not add (dependents are plugins and
  player providers, which only a user with `CONFIG_PROVIDERS_WRITE` may add).
- A dependent's row and its entry in the add dialog name the provider it needs.
- Loading, reloading, disabling and removing parents and dependents behave as today.
- No provider that is not a music source gains library sync, an access record or a place in the
  music source filters.

## Today (verified)

Dependency pairs in the tree, from the manifests:

| Parent → dependent | Parent | Dependent | Setup flow on dependent |
|---|---|---|---|
| hass → hass_players | plugin, single, admin | player, single | no |
| sonic_analysis → sonic_similarity | audio_analysis, single | plugin, single | no |
| opensubsonic → subsonic_scrobble | music, multi, member | plugin, single | no |
| sendspin → sendspin_source | player, builtin | plugin, builtin | no |
| sendspin → hue_entertainment | player, builtin | plugin, multi | yes |
| plex → plex_connect | music, multi, member | plugin, multi (one per MA player) | yes |
| yandex_music → yandex_ynison | music, multi, member | plugin, multi | yes |
| sendspin → _demo_sendspin_clients | dev only | dev only | no |

Server:

- `_load_provider` returns silently when the parent is not loaded (`mass.py:1436`);
  `load_provider_config` reloads the enabled dependents after a successful parent load
  (`mass.py:1002-1030`); `unload_provider` unloads dependents with the parent (`mass.py:1165`).
- `setup_provider` aborts with `already_configured` for a single-instance provider that has a
  config, and with `missing_dependency` when no enabled parent config exists
  (`controllers/config/flows.py:181-191`). A provider without `setup_flow.py` is created at once
  and the flow finishes in one step (`flows.py:194-203`).
- `_check_provider_setup_permission` lets a member add only a music provider that is
  multi-instance and self-service (`controllers/config/providers.py:910`). Every dependent in the
  table is a plugin or a player provider, so only an admin may add one.
- The finish handler of a provider flow returns `{"instance_id": ...}` and the FINISH step carries
  it as `result` (`models/setup_flow.py:347`).

Frontend:

- `AddProviderDialog.vue:284-313`: when `manifest.depends_on` names a domain without a *loaded*
  instance (`api.getProvider`), a confirm dialog offers the parent's setup flow instead, and the
  user has to come back for the dependent. Because the test is on loaded instances, a parent that
  is configured but down (for example the Home Assistant plugin while Home Assistant restarts)
  also triggers it, and the server then aborts the parent's flow with `already_configured`. This
  is the only place the frontend reads `depends_on`; the row of a dependent gives no hint.
- `SetupFlowDialog.vue`: a flow is launched through the `setupFlowDialog` event with
  `{kind: "provider", domain, onFlowEnded?}` (`plugins/eventbus.ts:116`). The FINISH screen has an
  "open settings" button for the created instance (`canOpenInstanceSettings`, line 448). The
  onboarding wizard's provider steps use the same add dialog and the same event
  (`ProvidersStep.vue:59`, `PlayersStep.vue:91`).
- `api.getProviderName(domain)` resolves a domain to its manifest name (used by the add dialog).

Base classes:

- `PluginProvider` re-declares 12 methods of `MusicProvider` (`models/plugin.py:114-527`: stream
  details, audio stream, search, similar tracks, recommendations, browse, playlist, radio, image).
  `MetadataProvider` re-declares 4 (`models/metadata_provider.py:67-148`). Controllers carry 14
  casts to `MusicProvider | PluginProvider` or the three-way union.
- Implementers of the overlap: plugins `radio_playlist`, `smart_playlist`, `recommendations`,
  `sonic_similarity`, `ai_radio`; metadata `musicbrainz`, `lastfm_recommendations`. The audio
  source plugins (`spotify_connect`, `airplay_receiver`, `sendspin_source`, ...) implement stream
  details and audio stream for their own `AUDIO_SOURCE` feature.
- Dispatch is already by feature: `get_providers_supporting_feature` collects by feature and uses
  the type only as tier order (`mass.py:672`).

## Architecture

### 1. Frontend: chained setup flows and the "Needs <parent>" hint

Parent first, then the dependent:

- The `setupFlowDialog` launch payload gains an optional `then: {domain}`.
- `AddProviderDialog.addProvider(B)` checks the parent through the configs, mirroring the server:
  an enabled config of `B.depends_on` exists. If it does, B's flow starts as today. If it does
  not, the dialog launches A's flow with `then: {domain: B}`; the confirm dialog goes. A parent
  without a setup flow finishes in one step, so the same path covers it.
- On a FINISH step whose launch carries `then`, the dialog launches the dependent's flow in place
  of the success screen. The dependent's own FINISH screen ends the chain. An ABORT of the parent
  ends the chain and shows the abort as today.

The dependents after a parent:

- On a provider FINISH step without `then`, the dialog lists the manifests whose `depends_on` is
  the finished domain, that are not configured yet or allow several instances, and that the user
  may add (`CONFIG_PROVIDERS_WRITE`). Each is a button with the dependent's name and its manifest
  description, next to "Open settings". A button launches that dependent's flow; a flow-less one
  finishes in one step and shows its own FINISH screen.
- The server enforces the permission regardless (`_check_provider_setup_permission`), so a client
  that gets the rule wrong is refused, not trusted.

The hint:

- `ProviderRow` shows "Needs <parent>" as the subtitle of a dependent, and the add dialog shows the
  same under the dependent's name, both from `depends_on` and `api.getProviderName`.
- Translations: the buttons and the hint; `provider_depends_on_confirm` is removed with the confirm
  dialog.

Nothing changes on the server for this section.

### 2. Server: capability mixins for the shared media methods

Four mixin classes in `music_assistant/models/media_capabilities.py`, holding the stubs that are
duplicated today, each stub raising `NotImplementedError` as now. Merged as
music-assistant/server#6667.

| Mixin | Methods | Mixed into |
|---|---|---|
| `MediaCatalogMixin` | `search`, `browse`, `get_playlist`, `get_playlist_tracks`, `get_radio`, `get_dynamic_radio_tracks` | Music, Plugin |
| `RecommendationsMixin` | `get_recommendations`, `get_recommendation_items` | Music, Plugin, Metadata |
| `MusicDiscoveryMixin` | `get_similar_tracks`, `get_similar_artists`, `get_artist_toptracks`, `get_artist_topalbums` (item-based) | Metadata, Plugin |
| `AudioStreamMixin` | `get_stream_details`, `get_audio_stream` | Music, Plugin |

- `MusicProvider`, `PluginProvider` and `MetadataProvider` inherit the mixins and drop their own
  copies. The mixins derive from `Provider`, so a controller variable typed by a capability still
  carries the provider's identity. `resolve_image` moves to `Provider` with return type
  `str | bytes | None`.
- The music provider keeps its own `get_similar_tracks`, `get_similar_artists`,
  `get_artist_toptracks` and `get_artist_topalbums`: they take provider item ids and keep a
  different contract from the item-based variants on the mixin. The name "discovery" alone
  collides with device discovery, hence `MusicDiscoveryMixin`.
- `delivers_normalized_audio` and `delivers_crossfaded_audio` stay on their classes: a definitive
  bool on the music provider, an optional hint (`bool | None`) on a plugin.
- The 14 union casts in the controllers become casts to (or `isinstance` checks against) the mixin.
  The feature check stays the gate, as today; the mixin only gives the methods one home and a
  type.
- No provider code changes and no behaviour changes. `tests/test_cross_type_features.py` keeps
  passing as it mocks the providers.

Rejected: extracting the media methods into "helper" music providers that depend on their parent.
A music provider gets library sync entries, an access record and member visibility rules, the
provider-mapping correction task and a place in every music source filter. None of the seven
implementers of the overlap is a music source: `recommendations` and `radio_playlist` are builtin
household features, `smart_playlist` and `sonic_similarity` compute over the library,
`lastfm_recommendations` and `musicbrainz` are metadata lookups, and the audio source plugins
stream a live input. Turning them into music providers would hide their rows from members
(`media/base.py:2476`, plugins are kept for every user because they carry no access record) and
widen their settings and code for no user value.

## Not doing

- Several types per provider, or a type derived from features (research #201).
- Automatic creation of a dependent when its parent exists, through a manifest flag or as a
  default provider on the add-on (decision 2).
- A "companions" line on the parent's provider card (decision 4).
- Helper music providers extracted from plugins or metadata providers (section 2).

## Phasing

Two sub-issues, independent of each other.

1. Frontend: chained setup flows for dependent providers, with the "Needs <parent>" hint.
2. Server: capability mixins for the media methods shared by music, plugin and metadata
   providers. Done: music-assistant/server#6667 (music-assistant/backlog#217).

## Risks and open points

- Chaining lives in the web frontend, which the mobile and desktop apps embed. Other API clients
  see the unchanged protocol and the `missing_dependency` abort.
- A chained dependent flow starts right after the parent's config is created; the parent may still
  be loading. The dependent's flow does not need the parent loaded (the server checks for an
  enabled config), and its instance waits for the parent as today.
- The mixin step changes the import graph of `music_assistant/models/`; a circular import between
  the mixins and the media item models is avoided by keeping the mixins free of controller
  imports, as the base classes are today.

## Verification

- Fresh install, add Plex Connect first: the dialog sets up Plex, then continues into Plex Connect
  without closing. Same from the onboarding wizard.
- Add Home Assistant MediaPlayers while the Home Assistant plugin is configured but down: the
  dialog does not offer to set up Home Assistant again.
- Add Plex as an admin: the finish screen offers Plex Connect. A member adding their own Plex
  account is not offered it.
- The Home Assistant MediaPlayers row and its add-dialog entry read "Needs Home Assistant".
- `pytest tests/test_cross_type_features.py tests/controllers/music` and
  `pre-commit run --all-files` clean after the mixin step.
