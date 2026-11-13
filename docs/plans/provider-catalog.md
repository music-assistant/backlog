# Catalog: technical plan

Status: proposal 2026-10-02, updated 2026-11-13. Owner: Marcel van der Veldt. This document is
the technical companion of the "Catalog: find and add everything Music Assistant can do" epic on
the project board. It is written so it can be fed to an agent for implementation; every claim
marked "verified" was checked in the referenced code (server at `5b53a4b79`, models at `dba5707`,
frontend at `7a636358`, add-on repo at `eeeb57c`) or in the referenced GitHub data on
2026-10-01. Where community providers live and how they are admitted, installed and updated is
covered by [community-catalog.md](community-catalog.md).

Terminology: "provider" is the code and contributor term and is used in this document. In the
app the words are "music sources", "player support", "metadata", "plugins" and "audio analysis";
the word "provider" does not appear in catalog copy.

## Summary

Finding something to add to Music Assistant (MA) means opening the right settings page and
pressing "add", which shows a plain list. Nothing tells a user what is new, what their network
could use, or who built and maintains what they are adding.

The catalog replaces the add dialog with one page that lists everything that can be added, for
every type, the way an app store does: sections for what is new in this release, what is
popular, what answered on your network, and what goes well with what you already have, then
everything else with search and filters. Every card credits the people who built and maintain
it. Built-in providers are always listed; community providers are listed with a Community badge
once the admin switched the community catalog on. An optional `support_url` lets a maintainer
receive reports where they prefer; without it, reports go to the support repository with the
right label preset, and the maintainer is mentioned on the issue, as Home Assistant does for
codeowners.

## Decisions taken with the maintainer (2026-10-01/02, 2026-11-13)

- The page is called **Catalog**. "Extensions" collides with the Plugins page and reads as
  third-party for built-in providers; "Marketplace" and "Store" collide with Home Assistant's
  Marketplace and add-on store, which most users see in the same settings tree.
- Built-in and community providers are shown together. Community providers appear only after
  the admin switched the community catalog on, with a Community badge. The maintainer is
  credited on the card and in a short note when adding one, together with where to ask for
  help.
- The catalog's recommendation sections are computed on the server and cover new, updated,
  popular, network-discovered and complementary providers, not only network discovery.
- Issues about a provider mention its codeowners. A maintainer may set `support_url` to receive
  reports elsewhere.

## Requirements

- One catalog page for all types. Each type page's "add" button opens the catalog filtered to
  that type. Member users keep today's restrictions (multi-instance and self-service providers
  only).
- Every card shows: icon, name, description, type, stage badge, the Community badge where it
  applies and who maintains it. The detail view adds credits, documentation, "Get help",
  "Request a change" and Add.
- A manifest may carry a `support_url`. "Get help" uses it when set, else the support
  repository with the provider label preset.
- Adding a community provider shows a short note crediting the maintainer and the place to
  ask for help.
- Recommendations: new in this release, updated in this release, popular, seen on the network
  but not set up, and complements of what is already configured.
- A "Request a provider" link to the feature request forum (worded "Missing something?").
- Issues labelled with a provider mention its codeowners once; a maintainer with a
  `support_url` is pointed to as well.

## Architecture

### Starting point (verified)

- Provider discovery is hard-wired to one root: `PROVIDERS_PATH` at `mass.py:122`, manifests
  parsed in `__load_provider_manifests` (`mass.py:1557-1609`), modules imported as
  `music_assistant.providers.<domain>` (`helpers/util.py:1464`). Folders starting with `_` load
  in dev mode only.
- `ProviderManifest` (models `provider.py:17-62`) has `type`, `domain`, `name`, `description`,
  `codeowners` (required), `stage`, `requirements`, `documentation`, `multi_instance`,
  `builtin`, `allow_disable`, `depends_on`, `icon`, `icon_images`, `mdns_discovery`,
  `upnp_discovery`, `has_setup_flow`, `self_service`, `credits`, and since backlog#205
  `support_url` and `tier`; the server sets `tier` from where a provider loads
  ([community-catalog.md](community-catalog.md), section 1). No version, no min/max server
  version. Unknown keys are ignored.
- `scripts/check_manifests.py` enforces the mandatory keys, `domain == folder`, a valid `type`
  and well-formed `codeowners`. 51 of 131 manifests list `@music-assistant`; 48 providers are
  owned only by people without write access.
- API: `providers/manifests` (`mass.py:496`) returns every manifest; `providers/icon` returns a
  base64 data URI; `providers` returns configured instances filtered per user.
- Frontend: `views/settings/AddProviderDialog.vue` is the add dialog. It filters out builtin,
  core, deprecated and configured single-instance providers, has search, a stage facet and
  stage badges (`helpers/provider_config.ts`), a hard-coded `POPULAR_PROVIDERS` list
  (`spotify, tidal, qobuz, filesystem_local, sonos, chromecast, airplay`), and handles
  `depends_on`. Codeowners and credits show only in `EditProvider.vue:128-141`. Settings
  navigation has separate entries per type (`Settings.vue:638-664`, query `?types=`), labelled
  "Music sources", "Player providers", "Metadata providers", "Plugins" and "Audio analysis"
  (`translations/en.json`). The `ProviderManifest` TypeScript interface is hand-written
  (`plugins/api/interfaces.ts:1576`).
- Discovery: the discovery controller browses the mDNS types of every manifest
  (`controllers/discovery/controller.py:251`) but only dispatches hits to loaded providers
  (`:342-344`); `_load_providers` (`mass.py:1349-1366`) auto-creates the `DEFAULT_PROVIDERS`
  that require an mDNS hit by checking the zeroconf cache. 13 public manifests (plus the demo
  player provider) declare mDNS or UPnP discovery.
- Support: `music-assistant/support` has an AI triage workflow (`triage.yml`, runs
  `ma_triage` with the bot app token) that applies provider labels; 67 labels exist, including
  per-provider ones. The bug report template has no provider dropdown. Nothing mentions
  codeowners today.
- Translations are compiled at build time from each provider's `strings.json`; manifest name
  and description are localized in `__post_serialize__`.

### 1. Catalog data and recommendations (server)

`providers/manifests` stays the list source for built-in providers. With the community catalog
switched on, community providers are added from the registry index
([community-catalog.md](community-catalog.md), section 6). Three additions, all additive;
recommendations cover built-in providers.

**Release metadata.** `catalog_meta.json` in the package: `{domain: {"added_in": "2.9.0",
"updated_in": "2.11.0"}}`, generated by `scripts/gen_catalog_meta.py` in the release workflow
before the wheel is built (first release tag containing the manifest's first commit; last
release tag containing the last commit under the provider folder; mass refactors count, since
telling them apart is not worth the cost). A dev checkout gets an empty file and no badges.
Exposed as `ProviderManifest.added_in` / `updated_in` (`str | None`), filled in
`__load_provider_manifests`.

**Recommendations.** `catalog/recommendations` (new command, `Scope.PROVIDERS_READ`) returns a
list of `{domain, reason, detail}` over providers the caller may add (same visibility rules as
the list), with reasons:

| reason | source |
|---|---|
| `new` | `added_in` equals the running release's minor version |
| `updated` | `updated_in` equals the running release's minor version and `added_in` does not |
| `discovered` | no config exists and an `mdns_discovery` type of the manifest is in the zeroconf cache, or an `upnp_discovery` target answered the one-off discovery cycle run at startup for unconfigured providers (reuses the cache check from `mass.py:1358-1362` and `_run_upnp_discovery_cycle(search_targets)`); `detail` carries the device name when known |
| `popular` | a curated constant `CATALOG_POPULAR` in `constants.py`, moved from the frontend's `POPULAR_PROVIDERS` |
| `complements` | a curated constant `CATALOG_PAIRINGS` in `constants.py`: `{configured_domain: [suggested_domain, ...]}`, for example `spotify` → `spotify_connect`, `filesystem_local` → `musicbrainz`, `fanarttv`, `lrclib`, a music source → `lastfm_scrobble`; `detail` names the configured provider |

Already configured single-instance providers never appear. The frontend groups by reason and
hides a recommendation the user dismissed (local storage). Both constants are edited per
release by the team; they are the editorial part of the store and deliberately not a remote
feed, so the recommendations add no network dependency.

**Help link.** Computed in the frontend: `support_url` when set, else a new-issue URL on the
support repository with the provider label preset
(`.../support/issues/new?template=1_bug_report.yml&labels=triage,<label>`, label from
`PROVIDER_SUPPORT_LABELS` or the domain). No server change.

### 2. Catalog page (frontend)

- New route `settings/catalog`, a "Catalog" card on the settings overview and a navigation
  entry. Subtitle: "Everything you can add to Music Assistant: music sources, player support,
  metadata and plugins, from the team and the community."
- Type names in the catalog and on the settings pages: "Music sources", "Player support"
  (today "Player providers"), "Metadata" (today "Metadata providers"), "Plugins", "Audio
  analysis". The two renames are part of this work; the word "provider" is not used in catalog
  copy. Internal keys stay.
- Layout, top to bottom, like a store front: "Found on your network" (`discovered`), "New in
  2.11" (`new`, with `updated` as a second row), "Popular" (`popular`), "Goes well with what
  you have" (`complements`), then the full list with search, type tabs and filters for stage
  and for built-in or community. Sections are hidden when empty.
- Cards show icon, name, one-line description, the stage badge where `shouldShowStageBadge`,
  and "by @owner" where a person is codeowner, else "by the Music Assistant team". Community
  providers also carry the Community badge. The detail panel shows the full description,
  "Created and maintained by @owner" (or the team), credits ("Made possible by", existing key),
  a Documentation button, "Get help" (help link above), "Request a change" (feature request
  forum) and Add. For a community provider it also shows the version and the install, update
  and remove actions.
- The type pages' add buttons route to the catalog with `?types=` preselected, carrying the
  member restrictions (`multiInstanceOnly`, `selfServiceOnly`) as query flags, replacing
  `AddProviderDialog.vue`. The dialog's filters (builtin, core, deprecated, configured
  single-instance, `depends_on` handling) move with it.
- Adding a community provider first shows a short note: "<Name> is created and maintained by
  @owner, a member of the community. For questions and problems, use <Get help link>." with
  Continue and Cancel. Shown each time; nothing persisted.
- "Missing something? Request it" at the bottom points at
  `https://github.com/orgs/music-assistant/discussions/categories/feature-requests-and-ideas`
  (already used by `About.vue:298`).
- Strings under `settings.catalog.*`; badge, note and section titles are translatable.
- Screenshots in the PR, per the frontend rules.

### 3. Maintainer mentions on issues (support repo)

- After `ma_triage` applies a provider label, a step maps label to domain (the inverse of
  `PROVIDER_SUPPORT_LABELS`, kept in the triage tool), reads the manifest from the server's dev
  branch, and comments once, mentioning the human codeowners (the `@music-assistant` handle is
  skipped): "This looks like it concerns <Name>, created and maintained by @owner." When the
  manifest has a `support_url`, the comment adds "The maintainer takes reports at <url>; you
  may want to continue there." The mention is the routing, there is no assignment. For a
  community provider the comment only points to the maintainer's support URL or repository;
  nobody is mentioned.
- The sticky triage comment links the provider's docs page.

### 4. Docs (`music-assistant.io`)

- New page "Catalog": what it lists, what the badges mean, who maintains community providers
  and how to reach them, how to request something.
- `community-extensions.md` links to the catalog and keeps listing cards and bridges.
- Contributor docs: `support_url`.

## Considered and not pursued

**Remote recommendation feed.** An editorial feed fetched from the project (featured items,
popularity from downloads) would make the store front live between releases. It adds a network
dependency and a privacy question for little gain while the constants are edited per release
anyway. Can be added behind the same `catalog/recommendations` command later.

## Phasing

1. Catalog page replacing the add dialog, type renames, maintainer credit and note, help and
   docs links.
2. `catalog/recommendations`, `catalog_meta.json`, the store-front sections.
3. Maintainer mention step in the triage workflow.
4. User docs.

The Catalog page follows the community catalog's own settings screens
([community-catalog.md](community-catalog.md), section 7) and then presents their data.

## Risks and open points

- `updated_in` counts mass refactors as updates. Acceptable for a badge; stated here so nobody
  tries to make it exact.
- `CATALOG_POPULAR` and `CATALOG_PAIRINGS` are editorial and need an owner who touches them
  per release; without that the store front goes stale. A release checklist item.

## Verification

- The catalog lists the same providers the add dialog listed for an admin and for a member,
  per type, and the type pages' add buttons land on the filtered catalog.
- A card of a provider with a person as codeowner shows "by @owner"; its "Get help" opens
  `support_url` when set and the labelled support issue form otherwise. Adding a community
  provider shows the note with the right names and link; adding a built-in provider does not.
- `catalog/recommendations` returns `discovered` for a provider whose mDNS type is in the
  zeroconf cache and no config exists, stops once configured, returns `complements` for a
  pairing whose left side is configured, and never returns a configured single-instance
  provider.
- A release build writes `catalog_meta.json`; the running release shows "New in <version>"
  for providers added in it.
- Support: an issue labelled with a provider gets exactly one mention comment, with the
  `support_url` line only when the manifest has one.
