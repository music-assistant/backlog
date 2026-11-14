# Marketplace: technical plan

Status: proposal 2026-10-02, updated 2026-11-14. Owner: Marcel van der Veldt. This document is
the technical companion of the "Marketplace: find and add everything Music Assistant can do" epic on
the project board. It is written so it can be fed to an agent for implementation; every claim
marked "verified" was checked in the referenced code (server at `5b53a4b79`, models at `dba5707`,
frontend at `7a636358`, add-on repo at `eeeb57c`) or in the referenced GitHub data on
2026-10-01. Where community providers live and how they are admitted, installed and updated is
covered by [community-catalog.md](community-catalog.md).

Terminology: "provider" is the code and contributor term and is used in this document. In the
app the words are "music sources", "player support", "metadata", "plugins" and "audio analysis";
the word "provider" does not appear in Marketplace copy.

## Summary

Finding something to add to Music Assistant (MA) means opening the right settings page and
pressing "add", which shows a plain list. Nothing tells a user what is new, what their network
could use, or who built and maintains what they are adding.

The Marketplace replaces the add dialog with one page that lists everything that can be added, for
every type, the way an app store does: sections for what is new in this release, what is
popular, what answered on your network, and what goes well with what you already have, then
everything else with search and filters. Every card credits the people who built and maintain
it. Built-in providers are always listed; community providers are listed with a Community badge
once the admin switched the community catalog on. An optional `support_url` lets a maintainer
receive reports where they prefer; without it, reports go to the support repository with the
right label preset, and the maintainer is mentioned on the issue, as Home Assistant does for
codeowners.

The Marketplace is for finding and adding. What is already set up is listed there as well, with a
jump to where it is managed: the settings pages per type stay, and remain the place to set up,
change and remove.

## Decisions taken with the maintainer (2026-10-01/02, 2026-11-13/14)

- The page is called **Marketplace** (decided 2026-11-14 with Paulus Schoutsen). It was called
  Catalog first (2026-10-02), because "Marketplace" and "Store" collide with Home Assistant's
  Marketplace and app store, which most users see in the same settings tree. That weighed less
  than a word Home Assistant users know: since 2026.11 Home Assistant has a Marketplace, and the
  direction of the Open Home Foundation is one marketplace that holds everything, built-in and
  community, with "community maintained" as a flag. Music Assistant's Marketplace already works
  that way: everything in one page, and the community catalog as an option on it.
  "Extensions" collides with the Plugins page and reads as third-party for built-in providers.
  Only what a user reads changes: the code keeps the word catalog (`Catalog.vue`, the
  `catalog/...` commands, the translation keys).
- Built-in and community providers are shown together. Community providers appear only after
  the admin switched the community catalog on, with a Community badge. The maintainer is
  credited on the card and in a short note when adding one, together with where to ask for
  help.
- The Marketplace's recommendation sections are computed on the server and cover new, updated,
  popular, network-discovered and complementary providers, not only network discovery.
- Issues about a provider mention its codeowners. A maintainer may set `support_url` to receive
  reports elsewhere.
- The Marketplace is for discovery, the settings pages per type are for managing, and those pages
  stay (2026-11-14). The Marketplace also lists what is installed, with a jump to the page where
  it is managed, shows which installed community providers have an update and which were
  updated recently, and links to each type page.

## Requirements

- One Marketplace page for all types. Each type page's "add" button opens the Marketplace filtered to
  that type. Member users keep today's restrictions (multi-instance and self-service providers
  only).
- The Marketplace lists what is already set up as well: an Installed view with every configured
  provider, and in the full list a card that says so. Both jump to the provider on its type
  page. The Marketplace links to each type page.
- With the community catalog on, the Marketplace shows which installed community providers have an
  update and which community providers are new or were updated recently.
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

### 1. Marketplace data and recommendations (server)

`providers/manifests` stays the list source for built-in providers. With the community catalog
switched on, community providers are added from the registry index
([community-catalog.md](community-catalog.md), section 6). Three additions, all additive.
Recommendations cover built-in providers; `new` and `updated` also cover community providers.

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

**Community providers.** With the community catalog on, `new` and `updated` also return
community providers, from the dates in the registry index: `new` when the provider entered the
community catalog in the last 30 days, `updated` when its newest version was released in the last 30
days and it is not `new` ([community-catalog-server.md](community-catalog-server.md), 2.2 and
2.12). They do not follow server releases, so they are computed when the command is called.

**Help link.** Computed in the frontend: `support_url` when set, else a new-issue URL on the
support repository with the provider label preset
(`.../support/issues/new?template=1_bug_report.yml&labels=triage,<label>`, label from
`PROVIDER_SUPPORT_LABELS` or the domain). No server change.

### 2. Marketplace page (frontend)

- New route `settings/marketplace`, a "Marketplace" card on the settings overview and a navigation
  entry. Subtitle: "Everything you can add to Music Assistant: music sources, player support,
  metadata and plugins, from the team and the community."
- Type names in the Marketplace and on the settings pages: "Music sources", "Player support"
  (today "Player providers"), "Metadata" (today "Metadata providers"), "Plugins", "Audio
  analysis". The two renames are part of this work; the word "provider" is not used in Marketplace
  copy. Internal keys stay.
- Layout, top to bottom, like a store front: "Found on your network" (`discovered`), "New in
  2.11" (`new`, with `updated` as a second row), "Popular" (`popular`), "Goes well with what
  you have" (`complements`), then the full list with search, type tabs and filters for stage
  and for built-in or community. Sections are hidden when empty.
- An Installed view next to the store front: every configured provider, built-in and community,
  and every installed community provider that has no configured instance yet, with the type
  visible and a jump to the provider on its type page. With the community catalog on it starts
  with the community providers that have an update, with Update and Update all, followed by
  those that were updated recently.
- In the full list a provider that is already set up says so and offers the jump to its type
  page instead of Add; Add stays when more than one instance is allowed. A link to the type
  page sits at the top of each type tab.
- The Marketplace finds and adds, and for community providers installs and updates. It does not
  configure: settings, enabling, disabling and removing stay on the type pages.
- Cards show icon, name, one-line description, the stage badge where `shouldShowStageBadge`,
  and "by @owner" where a person is codeowner, else "by the Music Assistant team". Community
  providers also carry the Community badge. The detail panel shows the full description,
  "Created and maintained by @owner" (or the team), credits ("Made possible by", existing key),
  a Documentation button, "Get help" (help link above), "Request a change" (feature request
  forum) and Add. For a community provider it also shows the version and the install and
  update actions.
- A switch to also show the community catalog, with its confirmation
  ([community-catalog.md](community-catalog.md), section 6).
- The type pages' add buttons route to the Marketplace with `?types=` preselected, carrying the
  member restrictions (`multiInstanceOnly`, `selfServiceOnly`) as query flags, replacing
  `AddProviderDialog.vue`. The dialog's filters (builtin, core, deprecated, `depends_on`
  handling) move with it; a configured single-instance provider is no longer left out but
  shown as set up.
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

- New page "Marketplace": what it lists, what the badges mean, who maintains community providers
  and how to reach them, how to request something.
- `community-extensions.md` links to the Marketplace and keeps listing cards and bridges.
- Contributor docs: `support_url`.

## Considered and not pursued

**Remote recommendation feed.** An editorial feed fetched from the project (featured items,
popularity from downloads) would make the store front live between releases. It adds a network
dependency and a privacy question for little gain while the constants are edited per release
anyway. Can be added behind the same `catalog/recommendations` command later.

## Phasing

1. Marketplace page replacing the add dialog, type renames, maintainer credit and note, help and
   docs links.
2. `catalog/recommendations`, `catalog_meta.json`, the store-front sections.
3. Maintainer mention step in the triage workflow.
4. User docs.

The community catalog has no screens of its own: its switch, list, install and update sit on
the Marketplace page ([community-catalog.md](community-catalog.md), section 7), so phase 1 comes
before the frontend step of the community catalog.

## Risks and open points

- `updated_in` counts mass refactors as updates. Acceptable for a badge; stated here so nobody
  tries to make it exact.
- `CATALOG_POPULAR` and `CATALOG_PAIRINGS` are editorial and need an owner who touches them
  per release; without that the store front goes stale. A release checklist item.
- Open: where an installed community provider is removed. Removing a configured instance is
  managing and stays on the type page. Removing the installed code has no place there while no
  instance is configured; the Installed view is the candidate. To settle with the screens.

## Verification

- The Marketplace offers Add for the same providers the add dialog listed for an admin and for a
  member, per type, and the type pages' add buttons land on the filtered Marketplace.
- A configured provider appears in the Installed view, and its card in the full list jumps to
  it on its type page; a single-instance provider that is set up offers no Add.
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
