# Community catalog: technical plan

Status: proposal 2026-11-13, updated 2026-11-18 to match what is built. Owner: Marcel van der Veldt. This document is
the technical companion of the "Community catalog" epic on the project board. It is written so
it can be fed to an agent for implementation; every claim marked "verified" was checked in the
server code at `7d5dd8e34` or in the referenced GitHub data on 2026-11-13. The Marketplace page that
presents built-in and community providers together is covered by
[provider-catalog.md](provider-catalog.md).

Terminology: "provider" is the code and contributor term. A **built-in** provider ships in the
server package. A **community provider** lives in its maintainer's own repository. The
**registry** is the repository that lists community providers and publishes the index that
servers read. In the app the collection is called the **community catalog**. A **channel** is
what a server follows: the nightly, the beta, or one per stable minor version such as stable 2.11.

## Summary

Music Assistant ships every provider inside the server, so every provider is reviewed,
documented and supported by a small team. Between November 2025 and October 2026 the pull
requests from outside the team went from 52 to 221 a month. Since November 2025, 105 pull
requests proposed a new provider, 78 of them from outside the team. Maintainers wrote 1,014
reviews and comments on those 105, and 165 of these went into pull requests that were never
merged. A new provider from outside the team waits a
median of 13.5 days for a merge and 65 days at the 75th percentile. Niche providers are declined
because everything built-in is something the team supports for years. The people behind them
ask for a place of their own, and the first unofficial loaders have appeared.

The community catalog is that place. A community provider lives in its maintainer's repository,
with its own documentation, support and release rhythm. The project owns a registry that lists
which providers exist. A provider is admitted once, after automated checks and one human
decision about what it does. From then on every release the maintainer publishes is checked
automatically and reaches users without anyone on the team looking at it. A server shows the
community catalog only after its admin switches it on, installs only what the registry lists,
and stops loading anything the registry blocks.

Built-in providers stay as they are. The server repository stays open to new providers, with the
requirements written down: real demand, documentation, the code standards and a maintainer who
keeps following up.

## Decisions taken with the maintainer (2026-11-13/14)

- **Two places.** Everything in the server repository is built-in, whether the team or a
  community developer maintains it. The community catalog is fully community-driven: own
  repository, support, documentation and releases.
- **Registry at the project, code at the maintainer.** The registry holds the list and the
  generated index, no provider code. The server downloads the exact commit the registry pinned
  from the maintainer's repository.
- **The index is the only source.** No custom repository URLs and no folder to drop a provider
  into. A local folder loads only in the server's development mode.
- **A kill switch is a hard requirement.** An entry the registry blocks is not loaded by any
  server.
- **A person decides once, about the concept.** Automated checks run first. A person then
  decides whether what the provider does is acceptable, not whether the code is good. Updates
  after admission are unattended.
- **Red lines instead of the full usage policy.** Being listed is not an endorsement and the app
  says everything in the community catalog is at the user's own risk. See section 5.
- **Authors act in good faith.** The checks catch drift and mistakes. Nothing is built to stop
  an author who hides things on purpose; the kill switch covers what surfaces later.
- **Compatibility is measured, not declared.** The registry tests every version against each
  supported server line and writes one index per server line.
- **The server's dependencies always win.** A version whose requirements clash with the server's
  is rejected by the registry.
- **One Marketplace, community shown on request.** The Marketplace has a switch to also show the
  community catalog. Switching it on asks for an explicit confirmation with the own-risk text.
  Updates are installed by the admin, with one switch to update the whole community catalog
  automatically.
- **The Marketplace is for discovery, the settings pages per type for managing** (2026-11-14). The
  Marketplace also shows what is installed, what has an update and what was updated recently. See
  [provider-catalog.md](provider-catalog.md).
- **A provider that moves out of the server takes its users along.** A server that already uses
  it fetches it from the registry after the upgrade.
- **Built-in or community follows from where a provider loads from.** A manifest cannot claim
  it. The Community badge is only for the community catalog.
- **The server repository stays inviting.** No "community catalog by default". The requirements
  for a built-in provider are explained in the READMEs, the developer docs and an automatic
  comment on every new-provider pull request.
- **A built-in provider without a maintainer can be moved by the team,** also when nobody takes
  it over. Those are parked together in one repository in the Music Assistant organisation.
- **The server part ships as soon as it is ready.**
- **No pilot.** The new-provider pull requests that are open today are the first admissions.
- **No new bot.** The team's review bot (Bot-MArley) reads and judges. Code from a provider only
  runs in workflows without secrets.
- **Plugin UI is declared, not injected.** See section 11.
- **No isolation, and nothing built in preparation for it.** See "Considered and not pursued".
- **Authors work in their own repository** with the server as a development dependency, and can
  test without a local development setup. See section 8.

## Decisions taken while building it (2026-11-15)

- **The community catalog is enabled and disabled,** like everything else in the app, not
  switched on and off.
- **The test fixture ships with the server.** A provider's tests get a started server through
  `pytest_plugins = ["music_assistant.testing"]`. See section 8.
- **The development image has its own workflow,** not a step of the release workflow: a
  release run stays as long as it is and cannot fail on it. The image is built on the exact
  nightly image and is only published after a build of it started. See section 8.
- **Moving providers starts with one.** The MCP Server (Community) plugin moves first, in 2.11,
  and the other eight providers of its maintainer follow once it has moved without trouble. The
  maintainer gets an issue that explains the new workflow and a ready-made pull request. See
  section 9.
- **A menu action declares what is shown, not what is called.** One command of the server runs
  the action on the provider instance the user chose; a provider registers no command of its
  own. Every entry shows which provider it comes from. See section 11.

## Decisions taken on 2026-11-18

- **A maintainer chooses to move.** A provider that is actively maintained moves to the
  community catalog only when its maintainer chooses to: the team may propose it, never force
  it. The submit form has a box for it. There is no separate pilot: the first move is one
  provider, watched over one nightly before others follow. This replaces "Moving providers
  starts with one" of 2026-11-15. See section 9.
- **Providers without an active maintainer** are parked in
  `music-assistant/unmaintained-providers`, after an announcement. Taking one over means moving
  it to an own repository. See section 9.
- **Releases are read in the registry's own pipeline,** with its own Claude API key in a
  workspace with a monthly spend limit, not by Bot-MArley: the verdict on a release reaches
  users without a person, so it must not depend on a bot's routine. Bot-MArley writes the
  summary at admission, where a person decides. This replaces "No new bot" for releases. See
  sections 3 and 4.
- **The core team decides admissions,** for now; this is looked at again with the comparison
  after about twenty admissions. See section 4.
- **No forms outside the settings.** Settings stay in the setup and config flows. A provider that
  wants an interface of its own serves a page, opened as a link to a short-lived address the
  server isolates, never in a frame. See section 11.

## Decisions taken on 2026-11-19

- **The reading signs in through GitHub, not with a key.** The release job gets a short-lived
  Claude token through workload identity federation (GitHub OIDC), so no key is stored anywhere.
  The workspace keeps its monthly spend limit. This replaces "its own Claude API key" of
  2026-11-18. See section 3.
- **The findings of a hold go to the maintainer, not into a security advisory.** A hold is
  reported in the provider's issue on the registry, with the rule, the file and the line. The
  project does not take on security scans of other people's code: what it can do is hold back a
  release or block a provider. The registry takes no private security reports either: a security
  problem in a provider is a matter for its maintainer. See sections 3 and 5.

## Requirements

- A server makes no request to the registry while the community catalog is off and no community
  provider is installed.
- The community catalog can only be switched off while nothing the admin installed from it is
  still installed. A provider the server installed by itself because it moved does not count.
- A server installs a community provider only when the index of its own channel lists that
  exact version, and only after the downloaded files match the content hash in the index.
- A blocked provider or version is not loaded. Its settings and library items are kept.
- A community provider cannot take the domain of a built-in provider, and cannot present itself
  as built-in.
- A community provider's requirements never change a package the server itself depends on.
- After a server upgrade, an installed community provider that is not listed for the new channel
  is switched to a listed version, or is not loaded and says why.
- A maintainer releases by publishing a GitHub release. Nobody on the team acts on it.
- A failed or held release produces an issue for the maintainer with the reason.
- Admission asks one decision from one person, on one screen.
- A settings entry whose provider is not available is visible and can be removed.
- An author can develop and test a community provider on Windows, macOS or Linux without a local
  Python setup.

## Architecture

### Starting point (verified)

- Provider discovery is hard-wired to one folder: `PROVIDERS_PATH` (`mass.py:127`), scanned in
  `mass.py:1643-1650`. Modules are imported as `music_assistant.providers.<domain>`
  (`helpers/util.py:1522-1526`), setup flows from the same package
  (`controllers/config/flows.py:691`). `music_assistant/providers` is a regular package.
- Requirements are installed at first load with `uv pip install --no-cache` into the server's
  own environment (`helpers/util.py:1446-1449`, `1530-1554`). There is no constraints file. The
  environment lives in the image and is not kept across image updates; only `/data` is a volume.
  The final image has no compiler, so only wheels for Python 3.14 on amd64 and arm64 install.
- Translations are compiled at build time from each provider's `strings.json`
  (`scripts/build_translations.py`); `strings.json` is not read at runtime. Icons are read from
  the provider folder wherever it is (`helpers/images.py:825`).
- A settings entry whose manifest is missing is skipped silently
  (`controllers/config/providers.py:122-124`) and cannot be removed, because removal looks the
  manifest up first (`providers.py:492`).
- Development mode is on when `PYTHONDEVMODE=1` or when a `.venv` folder sits next to the
  package, as in a source checkout (`mass.py:267-270`). Safe mode loads only `builtin` providers
  (`mass.py:388-393`).
- The server derives its channel from its version: `0.0.0` or `.dev` is nightly, `b` or `rc` is
  beta, anything else is stable (`controllers/dashboard/controller.py:419-426`).
- The `tier` field (`core` or `community`, default `community`) and `support_url` exist in the
  models. `tier` is written in every manifest and checked in pre-commit
  (`scripts/check_manifests.py:154-168`); the loader trusts the file. Both are in the nightly
  builds only, not in 2.11.0b8 or any 2.10 release. 30 of the 87 providers marked `community`
  belong to people with write access to the server repository.
- Core imports ten providers itself (`musicbrainz`, `universal_group`, `hass`,
  `universal_player`, `sync_group`, `builtin`, `sendspin`, `recommendations`, `radio_playlist`,
  `filesystem_local`), and 19 providers are imported by other providers. These cannot leave the
  package on their own.
- In six months about 21 changes to shared code also had to change provider code (commits that
  touched shared code and at least five providers; a heuristic count), with a peak of 8 in one
  month. Each was fixed for every provider in the same commit.
- A minor release spends about ten weeks in beta (2.10: 9.5 weeks; 2.11: 11 weeks so far), with
  a nightly build every day.
- The server does not fetch any update or registry information from a project endpoint today.
  It does have a precedent for a consented download with a recorded sha256, an atomic swap and a
  rollback (`providers/spotify_connect/soloist/runtime.py`).
- The server is not published on PyPI. The Dev app installs it from a repository reference
  (`server_repo`).

### 1. Built-in or community follows from where a provider loads from

- The loader sets `tier` itself: `core` for everything under `music_assistant/providers`,
  `community` for everything loaded from the community folder (section 6) or, in development
  mode, from a local folder. It also forces `builtin` to false for community providers.
- `tier` is removed from the manifests in the server repository and
  `scripts/check_manifests.py` rejects the key. `support_url` stays.
- Frontend: the Community badge keeps reading `tier`, so it now appears only for community
  providers. The maintainer line ("maintained by @owner") is shown for every provider that names
  a person as codeowner, built-in or community.
- `DEVELOPMENT.md` drops the tier rule from the manifest table.

This lands before 2.11.0 so the earlier meaning of the field never reaches a stable release.

### 2. The registry

A new repository in the Music Assistant organisation. It contains no provider code.

```
providers/<domain>/provider.json            written once, at admission
providers/<domain>/versions/<version>.json  generated per release
site/v1/<channel>.json                      generated, what servers read
```

- `provider.json`: `domain`, `repository` (`owner/name`), the folder inside it and, for a
  repository that holds several providers, the tag prefix of its releases, `maintainers`
  (GitHub handles), `status` (`listed`, `withdrawn` or `blocked`, with a reason and a link) and
  the admission record (who decided, when, which pull request).
- `versions/<version>.json`: the release tag, the full commit SHA, a sha256 over the provider
  folder's files, a snapshot of the manifest, the requirements, and the result of every check
  per channel.
- `site/v1/<channel>.json` (`site/v1/2.11.json` for stable 2.11, `site/v1/beta.json`,
  `site/v1/nightly.json`): for every listed provider the manifest data the app shows (name,
  description, type, icon, maintainers, documentation, support URL), the date it entered the
  catalog, the newest version that passed for this channel, the last five versions that
  passed, and per version the repository, commit, content hash, requirements and release date.
  It also carries the blocked and withdrawn entries.
- The index files are published as static files behind a CDN on a project domain. A server
  reads one file, once a day, with an ETag and a random offset. No server calls the GitHub API.
- The channels, listed in `channels.json` with the server image of each, are the current
  stable version, the beta and the nightly. When a new stable version appears, the index of the
  previous one is frozen: its versions are no longer
  recomputed, and blocked and withdrawn entries are still written to it.

### 3. Checks on every release

A workflow in the registry runs every night and on request, in separate jobs that share only
files.

**Fetch**, with the registry's own token: read the releases of every listed repository in
batches and download each new one at its commit. When a provider has several new releases
since the last run, only the newest is handled. GitHub pre-releases are ignored; a version
meant for the nightly only is a normal release.

**Check**, without any secret, because it executes code from the provider:

1. **Manifest.** Valid, the domain equals the admitted one, the version was not accepted
   before, no clash with a built-in domain.
2. **Dependencies.** For each channel the requirements are resolved against that server's
   pinned requirements as constraints. A clash, or a package without a wheel for Python 3.14 on
   amd64 and arm64, rejects the version for that channel.
3. **Load test.** In the server image of each channel: install, import the module and parse the
   manifest.

**Read**, by Claude, signed in through GitHub OIDC, in a step that never runs code:

4. **Red lines.** The difference with the last accepted version is read against section 5. The
   verdict is pass or hold. Only a hold goes to a person; its findings go to the maintainer in
   the issue of point 6.

**Write**, by the Music Assistant Bot app:

5. The version file, and the index of every channel the version passed for.
6. For a failed or held version: one issue per provider on the registry, with the label
   `release-check` and the output, that mentions its maintainers.

Every night the newest listed version of each provider is tested again against the current
image of every channel. A provider that stops loading on a channel falls back to the last version
that still passes there, or leaves that index, and the maintainer gets the issue with the
failing output. With a ten week beta, a maintainer usually knows weeks before a stable release.

A maintainer who wants to support the stable channel and follow the nightly at the same time
releases two versions; each ends up in the index it passes for.

Not built yet: rejecting a version with a known vulnerability, and running the provider's own
tests in the check step.

### 4. Admission

- The maintainer submits the provider with the form, and the Music Assistant Bot app opens the
  pull request that adds `provider.json`. The checks of section 3 run on
  the latest release, plus: the repository is public and has an open source licence, the author
  of the issue (the form) owns the repository or is a code owner in its manifest, and the domain
  is free.
- Bot-MArley posts one summary: which service it talks to, how audio reaches the server,
  whether an account is needed, the dependencies, the hosts it contacts, and its verdict on the
  red lines.
- A person of the core team approves or declines on that summary. The Music Assistant Bot app
  merges.
- The first twenty or so admissions are all decided by a person. The decisions are then compared
  with the bot's verdicts; if they agree, admissions the bot calls clear are merged without a
  person.
- A change of repository or maintainers is a pull request on `provider.json` and gets the same
  decision.
- Proposed, to keep the queue short: a pull request with failing checks is closed after two
  weeks, and an author has one open admission at a time.

### 5. Rules for the community catalog

Being listed means a provider did not cross a red line. It does not mean the project reviewed,
tested or supports it, and the app says so where the admin switches the community catalog on.

A provider is declined at admission, and blocked when found later, when it crosses one of the red
lines of the [community catalog rules](https://github.com/music-assistant/.github/blob/main/COMMUNITY_CATALOG_RULES.md). That page also says
what is allowed at the user's own risk, how to report a provider and what happens to a blocked
one.

### 6. Server: installing and loading

- **Switch.** The Marketplace is always there for everyone; what an admin switches on is the
  community catalog. It is off by default, admin only, and switching it on asks for an explicit
  confirmation with the own-risk text. A banner on the Marketplace page lets an admin discover it
  (switch on, or do not show again); the lasting control is in the page's top right menu, with
  the other options of the community catalog. A second setting updates everything in it
  automatically, off by default. Switching the community catalog off is refused while
  something the admin installed from it is still installed; a provider the server installed by
  itself because it moved does not count.
- **Index.** While the switch is on or a community provider is installed, the server fetches
  the index of its channel at start and once a day, and keeps the last copy. It picks the
  file from its own version: the nightly, the beta, or its stable minor version.
- **Install.** Download the archive of the pinned commit from GitHub, take the provider folder,
  compare the content hash, and move it into `<data>/community_providers/<domain>` with an
  atomic swap. The previous version is kept for a rollback.
- **Dependencies.** The requirements of all installed community providers are resolved
  together, with the server's own pinned requirements as constraints, wheels only, into one
  folder under the data directory. They survive image updates and cannot replace a package of
  the server. An install or update that does not resolve together with what is installed is
  refused and names the provider it conflicts with. The set is resolved again when the server
  version changes.
- **Load.** The community folder is added as a second location of the `music_assistant.providers`
  package, so providers import as they do today. A folder loads only when the installed domain,
  version and content hash match an entry of the index that is not blocked. When the index
  cannot be fetched at start the server uses its last copy; right after an upgrade to another
  channel that copy is of the previous channel, and installed versions load unless it blocks
  them. A domain that exists built-in is refused. Safe mode loads no community provider.
- **Development mode.** Only in development mode the server also loads providers from a local
  folder given at start, and the app shows that it runs in development mode.
- **Daily check.** A blocked provider is unloaded with the reason. An installed version that is
  not listed for this channel is switched to the newest listed one, or not loaded when none
  exists. A newer version is offered, or installed when automatic updates are on. A new version
  that fails to load is rolled back to the previous one when that is still listed and not
  blocked; otherwise the provider is not loaded and says why.
- **Moved providers.** The server carries the list of domains that moved out of it. At start, a
  settings entry for such a domain without an installed provider triggers the install, also
  when the switch is off; the rest of the community catalog stays hidden. The admin sees once
  that the provider is now maintained by its community maintainer.
- **Entries without a provider.** A settings entry whose provider is not available is shown as
  unavailable and can be removed, with the same library cleanup as today.
- **Texts.** A community provider's `strings.json`, and optional translation files next to it,
  are merged at load.
- **API.** Admin-only commands to list the community catalog with install state, and to install,
  update and remove.

### 7. Frontend

- The switch on the Marketplace page ([provider-catalog.md](provider-catalog.md), section 2), with
  its confirmation.
- With the switch on, community providers appear in the Marketplace, with the Community badge, the
  maintainer, a link to the provider's own documentation and "Get help" pointing at the
  maintainer's support URL or repository.
- The Marketplace shows which installed community providers have an update and which community
  providers are new or were updated recently. Installed community providers show their version
  and an update action. Where one is removed is an open point in
  [provider-catalog.md](provider-catalog.md).
- Notices for a provider that was blocked, could not be loaded on this server version, or moved
  out of the server.
- The settings pages per type stay the place to manage what is set up; a community provider is
  managed there like any other.

### 8. Developer workflow

An author works in an own repository, not in a fork of the server.

- **Template repository.** Contains one provider (manifest, code, texts, icon, a first test), a
  development container and the CI workflow.
- **The server as a development dependency.** The template pins a server release and installs it
  from the tagged source, which gives type checking, completion and the test fixtures. The
  fixtures an out-of-tree test needs ship inside the server package: with `pytest_plugins =
  ["music_assistant.testing"]` a test gets a started server in development mode that loads the
  providers of the repository, announces nothing on the network and listens locally.
- **Running it.** In the development container the server starts in development mode with the
  repository as the local provider folder. No local Python, ffmpeg or operating system
  requirement beyond Docker.
- **On Home Assistant.** The Dev app gets a `provider_repo` option with the syntax of the
  existing `server_repo`, so an author or a tester runs a provider from a repository, branch or
  pull request.
- **On Docker.** A new dev image on top of the nightly image
  (`ghcr.io/music-assistant/server:dev`) takes the server, frontend and provider references from
  the environment (`SERVER_REPO`, `FRONTEND_REPO`, `PROVIDER_REPO`). The Dev app and the dev
  image share one reference parser, kept in the server repository. The image has a workflow of
  its own: it finds the current nightly image, builds the dev image on exactly that one and
  starts it with the template's provider, and only then builds and publishes it for both
  platforms. It runs after every release run, where it does nothing when the nightly did not
  change, and when the files of the image change.
- **Open: the Dev app's version.** The release run raises the version of the Dev app before the
  new dev image exists, so an app that builds on the dev image would be offered an update on
  the previous image. Either the dev image's workflow raises that version after it published,
  or the app keeps building on the nightly image and only takes the scripts from the server.
- **CI.** A reusable workflow, owned by the registry, runs the manifest, dependency and load
  checks of section 3 on the author's pull requests. The red-line reading happens only in the
  registry.
- **Release.** Publish a GitHub release.
- **Docs.** A page in the developer documentation: the two places and what each asks, the red
  lines, the template, testing on Home Assistant and on Docker, and what the nightly test
  reports.

### 9. Moving providers out of the server

A provider leaves the server in a normal release: its folder is removed and its domain is added
to the list of moved providers (`MOVED_PROVIDER_DOMAINS`) in the same change. Providers that core
imports stay. Providers that import each other move together.

**What keeps working.** A server of the release that drops the provider and that has a
settings entry of it installs the provider from the index of its own channel at start, by itself,
also with the community catalog disabled. The settings entries and library items stay, because
the domain stays. The app shows one notice that the provider is now maintained by its community
maintainer. From then on it is an ordinary installed community provider; one the admin removes
is never installed by itself again. Servers on earlier channels keep their built-in copy.

**Accepted gaps.** A server without internet access or in safe mode does not get a moved
provider until it reaches the index; until then the provider's settings entries say why.

**Timing.** Moves land in 2.11, once the community catalog is in the beta. The registry has an
index for stable 2.11 before 2.11.0: the channel is added to `channels.json` during the beta
with the beta image as its image, because `ghcr.io/music-assistant/server:2.11` only exists from
2.11.0, and its image is switched to that one with the release.

**Before the first folder is removed:**

1. The registry has an index for the stable channel (`channels.json` lists `2.11`).
2. The server's CI refuses a change that removes a provider folder or adds a domain to
   `MOVED_PROVIDER_DOMAINS` until the indexes of the channels the change reaches (nightly, beta
   and the channel of the next release from that branch, as the release workflow names it) offer the
   provider with a version that passed. It also refuses a
   moved domain that still has a folder, and a removed provider the registry lists as moving that
   is not in the list.
3. The registry checks that every requirement also has a wheel for arm64; the checks run on
   amd64.

**One move, step by step:**

1. The maintainer's repository is ready: the provider is level with the server's copy, a release
   passes the reusable `check-provider.yml`, and the translations from Lokalise are in the
   repository. The maintainer gets an issue that explains the new workflow and a ready-made
   pull request with the translations and what else is missing.
2. The maintainer submits it with the box "Moving from the server" of the submit form. The
   bot checks that the submitter is a code owner of the provider in the server, and the
   admission pull request adds the domain to `moving` in `reserved_domains.json`. A person of
   the core team admits it.
3. The indexes of the channels the move reaches list the provider.
4. One server pull request adds the domain to `MOVED_PROVIDER_DOMAINS` and removes the folder
   with its tests.
5. The page on music-assistant.io points to the maintainer's documentation.
6. The maintainer stops syncing the provider into the server.

While the server still ships a provider, the load check of the registry imports the release's
code ahead of the server's own copy, and fails a release whose import still ends up at the server's
copy. Its requirements still resolve against the server's packages, which hold the pins of the
server's copy, so a release keeps those pins until the move.

**Who decides.** A provider that is actively maintained moves only when its maintainer chooses
to; the team may propose it, never force it. The first move is one provider, watched over one
nightly before others follow. The providers below are developed and released from their
maintainer's own repositories; the move was proposed to their maintainer, who decides.

| Provider | Domain | Type | Repository |
|---|---|---|---|
| MCP Server (Community) | `fastmcp_server` | plugin | `trudenboy/ma-provider-mcp` |
| Zvuk Music | `zvuk_music` | music | `trudenboy/ma-provider-zvuk-music` |
| KION Music | `kion_music` | music | `trudenboy/ma-provider-kion-music` |
| Yandex Disk | `filesystem_yandex_disk` | music | `trudenboy/ma-provider-yandex-disk` |
| Yandex Station | `yandex_station` | player | `trudenboy/ma-provider-yandex-station` |
| Yandex Smart Home | `yandex_smarthome` | plugin | `trudenboy/ma-provider-yandex-smarthome` |
| MSX Bridge | `msx_bridge` | player | `trudenboy/ma-provider-msx-bridge` |
| Yandex Music | `yandex_music` | music | `trudenboy/ma-provider-yandex-music` |
| Yandex Music Connect (Ynison) | `yandex_ynison` | plugin | `trudenboy/ma-provider-yandex-ynison` |

Ynison depends on Yandex Music and moves with it.

**Providers without an active maintainer.** The server repository asks for a maintainer who
follows up. A provider counts as without an active maintainer when, for six months, none of its
code owners did anything on it (a change, a review, an answer to a report) while fixes or reports
came in, and none of them was active elsewhere in the organisation either; or when the account of
its code owner no longer exists. The team looks at this every six months, and decides per
provider between carrying it itself in the server (when people use it, or the team already does
its fixes) and parking it.

A provider that is to be parked is announced for one minor release: its stage becomes
`unmaintained`, the app shows that it looks for a maintainer, its code owners are named in the
pull request that sets the stage, and it gets one issue "looking for a maintainer". When nobody
takes it over, it moves into `music-assistant/unmaintained-providers`: one folder per provider,
each released on its own through the community catalog with its own tag prefix. The core team
merges fixes there, and no new features. This is code that shipped with the server before, so
hosting it is nothing new. Users who have it set up get it from the catalog by themselves, also
with the community catalog disabled, and keep their settings. A parked provider that stops
loading on a channel and that nobody repairs leaves the index of that channel.

**Taking a parked provider over.** Whoever wants it says so on its issue, and the core team
decides. The new maintainer puts the folder in a repository of their own and publishes a release
with the same domain and a higher version; a registry maintainer then points the listing to that
repository in one pull request (`repository`, `repository_id`, `owner_id`, `folder` and
`tag_prefix` in `provider.json`). Servers update to that release by themselves and keep their
settings.

Issues about a community provider that arrive in the support repository are pointed to the
maintainer's support URL.

**The provider folder.** The last part of `folder` is the provider's domain or `provider`: one
predictable layout for a repository with a single provider (`provider/`, as the first nine have
it) and one for a repository with several (a folder per domain). A server installs the folder
under the domain, the registry takes the domain from `provider.json`, and the manifest in the
folder must name that domain. A local development folder, the test fixture and the Dev app load
all three layouts as well (server#7946).

### 10. Contributing to the server

- The READMEs and `DEVELOPMENT.md` state what a built-in provider asks: demand that can be
  shown, a documentation pull request, the code standards with tests, and a named maintainer who
  follows up on issues. They also say that a provider without an active maintainer may move to
  the community catalog.
- A pull request labelled `new-provider` gets one automatic comment with the same text. Once
  the community catalog accepts admissions, the comment also points to it as the other route.

### 11. What a community provider can show in the app

- **Menu actions.** A provider declares up to ten actions in its manifest: an id, a label,
  the media types it is offered for, the scope a user needs, an icon name and whether it acts on
  a player. The app draws them in the menu of a media item, or with the ways to play it for an
  action on a player, one entry per instance of the provider, and shows under each entry which
  provider it comes from. Choosing one calls one command of the server
  (`providers/invoke_menu_action`) with the instance, the action, the item and the player. The
  server checks that the user may use that instance, holds the scope and may use the player,
  and then calls the provider, which answers nothing or what a settings action answers: a
  message, a page to open, or both. A provider registers no command of its own.
- **What a declared action can say.** The server checks every action when it reads a manifest
  and leaves a broken one out; the provider still loads. A label is one readable line of at
  most 40 characters, a translated one of at most 80, and never a text that breaks the line or
  reorders what is around it. The app shows a label only as text and draws an icon only from a
  plain icon name.
- **No forms outside the settings.** Settings stay in the setup and config flows.
- **An own page.** A provider can serve a page of its own. An action answers with a path inside
  it, and the server turns that into a short-lived address bound to the user and the provider
  instance, checks both again on every request and forces a sandbox on every response, so the
  page cannot reach the app's sign-in. The app opens it as a link in a new tab, never in a
  frame: a page inside the app could pass for the app. Under Home Assistant Ingress and over
  Remote Access the app opens it on the server's address in the home network.
- **Links and images.** A link a community provider answers to another site must be https, and
  the app shows no remote images in the text of any provider.
- **No provider JavaScript in the app.** A feature that has to be fully part of the interface
  belongs in the server and frontend repositories.

This comes after the community catalog works, and serves built-in plugins as well.

## Considered and not pursued

**One repository with all the code, a branch per channel.** Versions follow the server by
construction and the team can repair every provider in one change. Every update is then a pull
request to a repository of the project, and the project hosts the code. Kodi's official add-on
repository works this way, with a team member merging every update. Flathub restricted bot
merges in 2024 after misuse
(<https://docs.flathub.org/blog/linter-restricting-automatic-merge>).

**Hosting the packages ourselves.** Always available and immutable, at the price of
distributing the code and receiving the takedown requests. In practice those go to whoever
hosts the code. Pinning a commit and checking a content hash gives the integrity without the
hosting. A repository that disappears can no longer be installed; installed copies keep working.

**Letting everything in, with only a security check.** It avoids the impression of endorsement.
It also puts the providers that offer paid music for free in the project's own catalog, behind
its own install button, while the project depends on streaming services for API access.

**The full usage policy.** The policy has engineering rules, such as never writing audio to
disk, that are right for what the project ships and stricter than what "at your own risk" needs.

**Custom repositories, or a folder for providers.** Every route by which Kodi became associated
with piracy went through a source the user added. Its switch for unknown sources, off by
default, did not help because every tutorial starts by flipping it. Unofficial loaders for Music
Assistant exist outside the project and cannot be prevented; the project does not build one.

**New providers go to the community catalog by default.** The server repository should stay
inviting for people who commit to a provider. Stating the requirements is enough to filter.

**A pilot.** The design is built once. The open pull requests are the first real use.

**Running community providers in a separate process.** Investigated on 2026-11-13.
Subinterpreters are documented as unsuitable for security, and the models package does not
import in one because `orjson` refuses to load. WebAssembly does not fit providers with native
dependencies and sockets. A separate process with a narrow contract is realistic for music and
metadata providers and is large: 39 of the 58 community music providers use only the HTTP
session, the cache and task helpers, and for 48 of them the server already fetches the audio
from a URL itself. Player and plugin providers are too entangled with core. Not built, and
nothing is added to the manifest in preparation, because there is no use for it today.

**Signing the index.** The index is served over HTTPS from a project domain. A signature can be
added without changing the format.

## Parked: codeowners ask for a backport of their own bugfix

Codeowners merge updates to their own built-in provider themselves: the review bot approves a
clean pull request that stays inside one provider, and the codeowner merges it with a command
(`DEVELOPMENT.md` in the server, "Getting your changes merged"). A new provider is still
approved by a maintainer. What is parked is a backport box in the pull request template that a
codeowner may tick for an own bugfix; it is picked up when codeowners ask for it.

## Phasing

1. Built-in or community follows from where a provider loads from (section 1). Before 2.11.0.
2. The rules for the community catalog and for contributing to the server (sections 5 and 10).
3. The registry: repository, checks on every release, the index (sections 2 and 3).
4. Admission (section 4).
5. Server: install, load, update (section 6), as soon as it is ready. The pull request sequence
   is in [community-catalog-server.md](community-catalog-server.md).
6. Frontend (section 7), on the Marketplace page, which is built first
   ([provider-catalog.md](provider-catalog.md), phase 1).
7. Developer workflow (section 8).
8. Move the first providers (section 9).
9. Providers without an active maintainer (section 9).
10. What a community provider can show in the app (section 11).

Steps 3 to 7 can run side by side. Step 8 needs 3 to 6 in a release.

## Risks and open points

- **Community code runs inside the server** and can reach everything the server can. The
  admission decision, the pinned commit, the red-line reading of every release and the kill
  switch are the answer; they do not make it impossible.
- **What is behind the install button reflects on the project.** The red lines are checked on
  every release, not only at admission, and a blocked provider stops loading everywhere.
- **Community providers will break more often than built-in ones.** Changes to shared code are
  no longer repaired for them by the team. The nightly test and the beta period give maintainers
  the warning.
- **The admission queue can flood.** The admission queue of the Home Assistant Community Store
  had 955 open pull requests on 2026-11-13, the oldest from January. The checks run before a
  person looks, the decision is one screen, failing requests close by themselves and clear cases
  become automatic.
- **The load test proves that a provider imports**, not that it works. A changed method
  signature that only fails at runtime is caught only by the provider's own tests. The template
  ships a first test.
- **GitHub is the download source.** An outage delays installs and updates. Installed providers
  keep running.
- **Open: who decides admissions** besides the core team.
- **Open: how fixes to parked providers are merged and released.**

## Verification

- A manifest in the server repository with a `tier` key fails pre-commit. A built-in provider
  shows no Community badge; one with a person as codeowner shows the maintainer line.
- With the switch off and no community provider installed, the server makes no request to the
  registry.
- Switching the community catalog off is refused while a provider the admin installed is still
  installed, and works once those are removed. A moved provider the server installed by itself
  keeps working with the switch off.
- Installing a version that the index of the server's channel does not list is refused. A download
  whose content hash differs is refused and leaves the installed version in place.
- A provider marked blocked in the index is unloaded at the next daily check and at start, its
  settings and library items stay, and the admin sees the reason.
- A provider folder copied into the community folder by hand does not load. In development mode
  only the folder given at start loads.
- A community manifest that says `"tier": "core"` or `"builtin": true`, or uses a built-in
  domain, is loaded as a community provider or refused.
- A requirement that would change a package of the server rejects the version in the registry;
  the same install on a server leaves the server's package untouched.
- After an image update, an installed community provider loads without reinstalling its
  dependencies.
- A release that imports on the stable channel and fails on the nightly appears in the stable
  index only, and its maintainer gets one issue with the output.
- Upgrading a server that uses a moved provider keeps the provider working, with its settings
  and library items, and shows the notice once.
- A settings entry of a removed community provider is shown as unavailable and can be removed.
- A new-provider pull request on the server gets exactly one comment with the requirements.
- A repository made from the template passes the reusable workflow, runs in the development
  container, and loads in the Dev app through `provider_repo`.
