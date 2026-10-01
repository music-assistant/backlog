# Catalog and community providers: technical plan

Status: proposal 2026-10-02. Owner: Marcel van der Veldt. This document is the technical
companion of the "Catalog: find and add everything Music Assistant can do" epic on the project
board. It is written so it can be fed to an agent for implementation; every claim marked
"verified" was checked in the referenced code (server at `5b53a4b79`, models at `dba5707`,
frontend at `7a636358`, add-on repo at `eeeb57c`) or in the referenced GitHub data on
2026-10-01.

Terminology: "provider" is the code and contributor term and is used in this document. In the
app the words are "music sources", "player support", "metadata", "plugins" and "audio analysis";
the word "provider" does not appear in catalog copy.

## Summary

Music Assistant (MA) ships every provider inside the server package. Finding one means opening
the right settings page and pressing "add", which shows a plain list. Nothing tells a user what
is new, what their network could use, or who built and maintains what they are adding.
Contributors who maintain a provider wait a median of 20 days for a merge, and the busiest of
them run a parallel release flow from their own repositories, with their own issue trackers.

The catalog replaces the add dialog with one page that lists everything the server ships, for
every type, the way an app store does: sections for what is new in this release, what is
popular, what answered on your network, and what goes well with what you already have, then
everything else with search and filters. Every card credits the people who built and maintain
it. A `tier` field in the manifest marks whether the Music Assistant team or a community member
maintains it, and an optional `support_url` lets a maintainer receive reports where they prefer;
without it, reports go to the support repository with the right label preset, and the
maintainer is mentioned on the issue, as Home Assistant does for codeowners. Community providers
stay in the server repository and in the image. Their codeowners get a fast lane: a bot merges
updates to their own provider after CI and an AI review pass, and new providers need one human
policy sign-off. A dev container gives Docker users what the Dev add-on gives Home Assistant
users.

Nothing loads code from outside the package. Runtime installation of providers, custom
repositories, a custom providers folder and an out-of-process plugin API were investigated and
are not part of this plan; the reasons and the triggers to revisit are recorded below.

## Decisions taken with the maintainer (2026-10-01/02)

- The deliverable is discovery plus ownership, not a plugin loader. No third-party code is
  installed at runtime. Home Assistant's Marketplace (home-assistant/core#183502, merged
  2026-10-01 for 2026.11) is a lift-and-shift of HACS and is watched, not copied.
- Community providers live in-tree. The tier marks who maintains a provider; the admission bar
  (usage policy, quality) is the same as today. A separate repository was kept open as an
  option and rejected for now: breaking changes to the provider base classes happen several
  times a month and are fixed for every provider in one PR today; a sibling repo turns each of
  those into a two-step change.
- The page is called **Catalog**. "Extensions" collides with the Plugins page and reads as
  third-party for core providers; "Marketplace" and "Store" collide with Home Assistant's
  Marketplace and add-on store, which most users see in the same settings tree.
- Community providers are listed with everything else, with a Community badge. The maintainer
  is credited on the card and in a short note when adding one, together with where to ask for
  help. The copy puts the maintainer in the spotlight; it does not say "unsupported". No opt-in
  toggle: 48 providers that ship today would disappear from the list for existing users.
- The catalog's recommendation sections are computed on the server and cover new, updated,
  popular, network-discovered and complementary providers, not only network discovery.
- Issues about any provider mention its codeowners, core and community alike. A maintainer may
  set `support_url` to receive reports elsewhere.
- Codeowners may merge updates to their own provider after the bot's checks pass; new providers
  need a human policy sign-off, then the same bot path. The AI reviewer vendor is left open.
- Developers test out-of-tree work with the Dev add-on (exists) or a dev container (new). No
  custom providers folder in production installs.
- The out-of-process plugin API over the websocket is not pursued: it isolates secrets, not
  policy, and players and external sources already have out-of-process paths (Sendspin,
  external source sessions).
- Runtime install from an own CI-built catalog: not needed now, see "Considered and not
  pursued" for the triggers.

## Requirements

- One catalog page for all types. Each type page's "add" button opens the catalog filtered to
  that type. Member users keep today's restrictions (multi-instance and self-service providers
  only).
- Every card shows: icon, name, description, type, stage badge, tier badge and who maintains
  it. The detail view adds credits, documentation, "Get help", "Request a change" and Add.
- Core versus community is explicit in the manifest and validated in pre-commit.
- A manifest may carry a `support_url`. "Get help" uses it when set, else the support
  repository with the provider label preset.
- Adding a community provider shows a short note crediting the maintainer and the place to
  ask for help.
- Recommendations: new in this release, updated in this release, popular, seen on the network
  but not set up, and complements of what is already configured.
- A "Request a provider" link to the feature request forum (worded "Missing something?").
- Codeowners can get their own provider updates merged without a core maintainer, after CI,
  an AI review and the existing dependency review pass. New providers cannot be self-merged.
- Issues labelled with a provider mention its codeowners once; a maintainer with a
  `support_url` is pointed to as well.
- A published dev container runs a fork, branch, PR or commit the way the Dev add-on does.
- No change to what loads: the provider root stays `music_assistant/providers`, and nothing in
  this plan installs or imports code from elsewhere.

## Architecture

### Starting point (verified)

- Provider discovery is hard-wired to one root: `PROVIDERS_PATH` at `mass.py:122`, manifests
  parsed in `__load_provider_manifests` (`mass.py:1557-1609`), modules imported as
  `music_assistant.providers.<domain>` (`helpers/util.py:1464`). Folders starting with `_` load
  in dev mode only.
- `ProviderManifest` (models `provider.py:17-62`) has `type`, `domain`, `name`, `description`,
  `codeowners` (required), `stage`, `requirements`, `documentation`, `multi_instance`,
  `builtin`, `allow_disable`, `depends_on`, `icon`, `icon_images`, `mdns_discovery`,
  `upnp_discovery`, `has_setup_flow`, `self_service`, `credits`. No tier, no support URL, no
  version, no min/max server version. Unknown keys are ignored.
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
- CI and merge: the dev branch ruleset requires one approving review, resolved threads, green
  `lint` and `test`, squash via merge queue; org admins bypass. `auto-merge-dependency-updates.yml`
  approves with the Actions token and enqueues with a token minted from the "Music Assistant
  Bot" GitHub App (`vars.MUSIC_ASSISTANT_BOT_CLIENT_ID`, `secrets.MUSIC_ASSISTANT_BOT_PRIVATE_KEY`).
  `dependency-approval-command.yml` is the comment-command pattern (`/approve-dependencies`,
  maintainers only). `scripts/check_provider_scope.py` classifies changed paths into provider
  folders versus shared code (advisory today). `scripts/ci_test_scope.py` runs only the changed
  providers' tests. Copilot reviews every push.
- Support: `music-assistant/support` has an AI triage workflow (`triage.yml`, runs
  `ma_triage` with the bot app token) that applies provider labels; 67 labels exist, including
  per-provider ones. The bug report template has no provider dropdown. Nothing mentions
  codeowners today.
- Dev add-on: `music_assistant_dev` has no `image:` key, so the Supervisor builds it locally
  from `ghcr.io/music-assistant/server:nightly` plus build tools; `entrypoint.sh` installs the
  server from `server_repo` (`owner/repo@ref`, `pr-N`, branch or SHA) and the frontend from
  `frontend_repo`. There is no equivalent for Docker users.
- Translations are compiled at build time from each provider's `strings.json`; manifest name
  and description are localized in `__post_serialize__`.

### 1. Maintainer tier and support URL (models + server)

Models:

- `ProviderTier(StrEnum)` with `CORE = "core"` and `COMMUNITY = "community"` in `enums.py`.
- `ProviderManifest.tier: ProviderTier = ProviderTier.COMMUNITY`. The default is community so
  that a missing field never silently claims core.
- `ProviderManifest.support_url: str | None = None`. Where the maintainer wants reports; a
  repository issues page, a forum thread, a Discord invite.

Server:

- `scripts/check_manifests.py`: `tier` must be a valid value; `tier: core` requires
  `@music-assistant` in `codeowners` (the project owns it, the humans are listed next to it).
  Public providers may not omit the field, so the choice is made once per provider.
  `support_url`, when present, must be an `https` URL.
- One-time edit of the manifests the team considers core: add `"tier": "core"` and, where
  missing, `@music-assistant` to codeowners. Everything else gets `"tier": "community"`. The
  initial split is the maintainer's call; the 51 manifests already listing `@music-assistant`
  are the starting point. Codeowners of community providers are asked, in the PR, whether they
  want a `support_url`.
- The frontend interface gains `tier` and `support_url`. Additive fields, no
  `API_SCHEMA_VERSION` bump unless a non-bundled client needs to detect them (ask first).
- `DEVELOPMENT.md` manifest table documents both fields and the rule.

### 2. Catalog data and recommendations (server)

`providers/manifests` stays the list source. Three additions, all additive.

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
feed, so the server has no new network dependency.

**Help link.** Computed in the frontend: `support_url` when set, else a new-issue URL on the
support repository with the provider label preset
(`.../support/issues/new?template=1_bug_report.yml&labels=triage,<label>`, label from
`PROVIDER_SUPPORT_LABELS` or the domain). No server change.

### 3. Catalog page (frontend)

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
  and tier. Sections are hidden when empty.
- Cards show icon, name, one-line description, the stage badge where `shouldShowStageBadge`,
  and for community providers a Community badge plus "by @owner". Core cards say "by the
  Music Assistant team". The detail panel shows the full description, "Created and maintained
  by @owner" (or the team), credits ("Made possible by", existing key), a Documentation button,
  "Get help" (help link above), "Request a change" (feature request forum) and Add.
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

### 4. Community fast lane (server repo CI)

New workflow `community-merge.yml` on `issue_comment` (command `/merge`) and on
`pull_request_target` label `codeowner-merge`:

1. Scope: `git diff --name-only base...head` must classify, via `check_provider_scope.classify`,
   into exactly one provider and no shared files (generated files allowed). Changes to a
   manifest's `codeowners`, `tier` or `support_url` are shared changes for this purpose.
2. Ownership: the commenter must be listed in that provider's `codeowners` as read from the
   **base** branch manifest (so a PR cannot add its author). A PR whose provider has no base
   manifest is a new provider and is refused by this workflow.
3. Checks: required status checks `lint` and `test` green, the dependency security status not
   failing (so `/approve-dependencies` by a maintainer stays required when `requirements`
   change), zero unresolved review threads.
4. AI verdict: a review job runs the AI reviewer against the diff with the repo's standards
   files (`.github/copilot-instructions.md`, `.github/instructions/music-assistant-standards.instructions.md`,
   `DEVELOPMENT.md` provider rules) and must return "approve" as a check run. Copilot's PR
   review cannot be read as a branch-protection approval, so this is a separate step; the
   vendor (Copilot via a model call as in the support repo's triage, or Claude Code Action
   with an Anthropic key secret) is left open and the step is written to be swappable.
5. Merge: approve with the Actions token, enqueue with the bot App token, exactly as
   `auto-merge-dependency-updates.yml` does. Comment the outcome on the PR; on refusal, say
   which step failed.

New providers: PR template gains a usage policy checklist (fetches audio as an ordinary client,
no audio URL exposure, no stored decoded audio, throttled and cached, respects stream limits).
A maintainer who has checked it sets `policy-approved`; from then on the same workflow may run
on a maintainer's `/merge`. The human sign-off is the one step that stays, because the review
that caught a login grabber in a past provider PR was a human one.

`DEVELOPMENT.md` gets a "Community providers" section: what the tier means, what codeowners may
merge, and that a provider's codeowner is expected to fix it when a base-class change breaks it
(the core team keeps fixing all providers in refactor PRs, as today).

### 5. Maintainer mentions on issues (support repo)

- After `ma_triage` applies a provider label, a step maps label to domain (the inverse of
  `PROVIDER_SUPPORT_LABELS`, kept in the triage tool), reads the manifest from the server's dev
  branch, and comments once, mentioning the human codeowners (the `@music-assistant` handle is
  skipped): "This looks like it concerns <Name>, created and maintained by @owner." When the
  manifest has a `support_url`, the comment adds "The maintainer takes reports at <url>; you
  may want to continue there." Same wording for core and community; the mention is the
  routing, there is no assignment.
- The sticky triage comment links the provider's docs page.

### 6. Dev container (server + add-on repos)

- `Dockerfile.dev` in the server repo: `FROM ghcr.io/music-assistant/server:nightly`, the same
  apt packages as the Dev add-on, an entrypoint that reads `SERVER_REPO` and `FRONTEND_REPO`
  from the environment with the Dev add-on's reference syntax. Published as
  `ghcr.io/music-assistant/server:dev` by the nightly release job.
- The Dev add-on's `entrypoint.sh` and the new one share the reference parser; move it to a
  script in the server repo that both images copy, so the syntax cannot drift.
- Docs: "Test a provider on Home Assistant with the Dev add-on, on Docker with the dev image."

### 7. Docs (`music-assistant.io`)

- New page "Catalog": what it lists, what the badges mean, who maintains community providers
  and how to reach them, how to request something.
- `community-extensions.md` links to the catalog and keeps listing cards and bridges.
- Contributor docs: the fast lane, the policy checklist, `support_url`, the dev container.

## Considered and not pursued

**Runtime install from an own CI-built catalog.** Benefits: a fix reaches stable users the
same day instead of with the next weekly patch; heavy or unaudited dependencies stay out of the
image; the image is visibly core-only. Costs, all permanent: `/app/venv` is wiped on every image
update (`Dockerfile:109,127` makes it world-writable, nothing persists it), so code and packages
need a home under `/data` and re-validation after each update; the image has no compiler, so
only pure-Python or cp314 manylinux wheels for amd64 and arm64 install; `install_package` is a
bare `uv pip install --no-cache` without a constraints file (`helpers/util.py:1384-1391`), so a
pin can change core dependencies; translations are build-time only; a config whose manifest
vanished cannot be removed (`config/providers.py:478` raises `KeyError`) and its library rows
stay; there is no provider API version and the base classes changed 54 times (music provider)
and 141 times (player) in a year, so a per-server-version build matrix would be needed; plus a
removal list, kill switch, consent gate and update notifications. What exists already covers
most of the benefit: the fast lane lands in dev within hours and in the nightly that night, the
`bugfix` + `backport-to-stable` labels auto-backport, stable patches ship about weekly.
Revisit if a provider class appears whose dependencies the project refuses to ship, or if a
fragile provider proves weekly too slow; a provider-only hotfix release is then the cheaper
fix. The catalog UI is built so an Install button can be added without redesign.

**Out-of-process plugins over the websocket API.** The only real isolation (the API is
versioned, a remote contract is small and explicit), but it does not keep policy violators out:
a scraper works as a sidecar feeding audio over the API just as well. Players already run out
of process through Sendspin and audio sources through external source sessions. Music sources
over RPC would be a separate epic. Dropped.

**Custom providers folder.** A one-way door: it becomes the distribution channel for the
providers the usage policy rejects ("copy this folder"), which is how custom components started
in Home Assistant. The Dev add-on and the dev container cover testing.

**Separate community repository.** See the decisions section. Revisit together with runtime
install, where it becomes natural because the providers leave the image.

**Opt-in gate for community providers.** Rejected as a regression for the 48 providers that
ship today without one; the per-instance "unofficial API" alert entry (`CONF_ENTRY_UNOFFICIAL_PROVIDER`)
keeps covering the providers that need a warning.

**Remote recommendation feed.** An editorial feed fetched from the project (featured items,
popularity from downloads) would make the store front live between releases. It adds a network
dependency and a privacy question for little gain while the constants are edited per release
anyway. Can be added behind the same `catalog/recommendations` command later.

**HACS-style catalog with custom repositories.** Highest exposure. The usage policy rests on
what the project ships, accepts and supports; a loader with install-from-URL makes MA the
runtime for the scrapers and DRM bypasses that were rejected at PR time, and the stars in the
out-of-tree ecosystem sit almost entirely on exactly those (one YouTube Music scraper repo has
82 of them). In-process code also reads the Supervisor manager token, the HA admin token, every
decrypted user secret, the bundled service credentials and the Widevine CDM, and can register
unauthenticated routes. Not pursued.

## Phasing

1. Tier and support URL fields, validation, manifest edits, frontend interface (models +
   server + frontend).
2. Community fast lane workflow, PR template checklist, `DEVELOPMENT.md` section.
3. Catalog page replacing the add dialog, type renames, maintainer credit and note, help and
   docs links.
4. `catalog/recommendations`, `catalog_meta.json`, the store-front sections.
5. Maintainer mention step in the triage workflow.
6. Dev container and shared entrypoint parser.
7. User and contributor docs.

Steps 1 and 2 are independent of the UI and unblock contributors first. Steps 3 and 4 are the
user-facing part. 5 to 7 can run in parallel with 3 and 4.

## Risks and open points

- The initial core/community split is a judgment call per provider and will be read by
  contributors as a statement about their work. The copy credits the maintainer and never
  says "unsupported"; the docs explain that community means "built and maintained by a
  community member".
- A bot that merges on a comment is a new trust boundary. Mitigations: base-branch codeowner
  check, scope check that treats codeowner, tier and support URL edits as shared changes,
  required checks, dependency status, AI verdict, and the refusal to self-merge a new provider.
  Org admins can disable the workflow with one commit.
- `updated_in` counts mass refactors as updates. Acceptable for a badge; stated here so nobody
  tries to make it exact.
- `CATALOG_POPULAR` and `CATALOG_PAIRINGS` are editorial and need an owner who touches them
  per release; without that the store front goes stale. A release checklist item.
- The AI review vendor and its cost are open; the workflow is written so the verdict step is
  swappable.
- Whether the busiest external contributor moves development in-tree once the fast lane
  exists is for the maintainer to ask him.

## Verification

- `pre-commit run --all-files` fails for a manifest without `tier`, with an invalid value,
  with `tier: core` and no `@music-assistant` codeowner, or with a non-https `support_url`.
- The catalog lists the same providers the add dialog listed for an admin and for a member,
  per type, and the type pages' add buttons land on the filtered catalog.
- A community card shows "by @owner"; its "Get help" opens `support_url` when set and the
  labelled support issue form otherwise. Adding it shows the note with the right names and
  link; adding a core provider does not.
- `catalog/recommendations` returns `discovered` for a provider whose mDNS type is in the
  zeroconf cache and no config exists, stops once configured, returns `complements` for a
  pairing whose left side is configured, and never returns a configured single-instance
  provider.
- A release build writes `catalog_meta.json`; the running release shows "New in <version>"
  for providers added in it.
- Fast lane: a codeowner's `/merge` on an in-scope green PR merges; the same command from a
  non-codeowner, on a PR touching shared files, with a failing dependency status, with an
  unresolved thread, or on a new provider is refused with the reason in a comment.
- Support: an issue labelled with a provider gets exactly one mention comment, with the
  `support_url` line only when the manifest has one.
- Dev container: `docker run -e SERVER_REPO=pr-1234 ghcr.io/music-assistant/server:dev` runs
  that PR.
