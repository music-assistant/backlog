# Community catalog: technical plan

Status: proposal 2026-11-13, updated 2026-11-14. Owner: Marcel van der Veldt. This document is
the technical companion of the "Community catalog" epic on the project board. It is written so
it can be fed to an agent for implementation; every claim marked "verified" was checked in the
server code at `7d5dd8e34` or in the referenced GitHub data on 2026-11-13. The Catalog page that
presents built-in and community providers together is covered by
[provider-catalog.md](provider-catalog.md).

Terminology: "provider" is the code and contributor term. A **built-in** provider ships in the
server package. A **community provider** lives in its maintainer's own repository. The
**registry** is the repository that lists community providers and publishes the index that
servers read. In the app the collection is called the **community catalog**. A **server line**
is what a server follows: a stable minor line such as 2.10, the beta or the nightly.

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
- **One Catalog, community shown on request.** The Catalog has a switch to also show the
  community catalog. Switching it on asks for an explicit confirmation with the own-risk text.
  Updates are installed by the admin, with one switch to update the whole community catalog
  automatically.
- **The Catalog is for discovery, the settings pages per type for managing** (2026-11-14). The
  Catalog also shows what is installed, what has an update and what was updated recently. See
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
- **No pilot.** The first providers to move are those whose maintainer already develops and
  releases them from own repositories; the new-provider pull requests that are open today are
  the first admissions.
- **No new bot.** The team's review bot (Bot-MArley) reads and judges. Code from a provider only
  runs in workflows without secrets.
- **Plugin UI is declared, not injected.** See section 11.
- **No isolation, and nothing built in preparation for it.** See "Considered and not pursued".
- **Authors work in their own repository** with the server as a development dependency, and can
  test without a local development setup. See section 8.

## Requirements

- A server makes no request to the registry while the community catalog is off and no community
  provider is installed.
- Switching the community catalog off hides it and stops new installs. Installed community
  providers keep working until the admin removes them.
- A server installs a community provider only when the index for its own server line lists that
  exact version, and only after the downloaded files match the content hash in the index.
- A blocked provider or version is not loaded. Its settings and library items are kept.
- A community provider cannot take the domain of a built-in provider, and cannot present itself
  as built-in.
- A community provider's requirements never change a package the server itself depends on.
- After a server upgrade, an installed community provider that is not listed for the new server
  line is switched to a listed version, or is not loaded and says why.
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
index/<server line>.json                    generated, what servers read
```

- `provider.json`: `domain`, `repository` (`owner/name`), the folder inside it and, for a
  repository that holds several providers, the tag prefix of its releases, `maintainers`
  (GitHub handles), `status` (`listed`, `withdrawn` or `blocked`, with a reason and a link) and
  the admission record (who decided, when, which pull request).
- `versions/<version>.json`: the release tag, the full commit SHA, a sha256 over the provider
  folder's files, a snapshot of the manifest, the requirements, and the result of every check
  per server line.
- `index/<server line>.json` (`index/2.10.json` for a stable line, `index/beta.json`,
  `index/nightly.json`): for every listed provider the manifest data the app shows (name,
  description, type, icon, maintainers, documentation, support URL), the date it entered the
  catalog, the newest version that passed for this server line, the last five versions that
  passed, and per version the repository, commit, content hash, requirements and release date.
  It also carries the blocked and withdrawn entries.
- The index files are published as static files behind a CDN on a project domain. A server
  reads one file, once a day, with an ETag and a random offset. No server calls the GitHub API.
- Supported server lines are the current stable line, the beta and the nightly. When a new
  stable line appears, the index of the previous one is frozen: its versions are no longer
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
2. **Dependencies.** For each supported server line the requirements are resolved against that
   server's pinned requirements as constraints. A clash, a package without a wheel for
   Python 3.14 on amd64 and arm64, or a known vulnerability rejects the version for that line.
3. **Load test.** In the official image of each supported server line: install, import the
   module, parse the manifest, and run the provider's own tests where it has them.

**Read**, by Bot-MArley, which never runs code:

4. **Red lines.** The difference with the last accepted version is read against section 5. The
   verdict is pass or hold. A hold goes to a person; nothing else does.

**Write**, by the Music Assistant Bot app:

5. The version file, and the index of every server line the version passed for.
6. For a failed or held version: an issue on the provider's repository with the output, or on
   the registry when that repository has no issues.

Every night the newest listed version of each provider is tested again against the current
image of every supported server line. A provider that stops loading on a server line falls back to the last version
that still passes there, or leaves that index, and the maintainer gets the issue with the
failing output. With a ten week beta, a maintainer usually knows weeks before a stable release.

A maintainer who wants to support the stable line and follow the nightly at the same time
releases two versions; each ends up in the index it passes for.

### 4. Admission

- The maintainer opens a pull request that adds `provider.json`. The checks of section 3 run on
  the latest release, plus: the repository is public and has an open source licence, the author
  of the pull request owns or maintains it, and the domain is free.
- Bot-MArley posts one summary: which service it talks to, how audio reaches the server,
  whether an account is needed, the dependencies, the hosts it contacts, and its verdict on the
  red lines.
- A person approves or declines on that summary. The Music Assistant Bot app merges.
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

Declined at admission, and blocked when found later:

- music the user has no right to: getting around an account, a subscription tier or a region,
  and sources that offer music without the rights holders' permission;
- circumventing copy protection;
- downloading or exporting music to keep it (a cache used for playback is fine);
- harmful behaviour: sending the user's data elsewhere, loading code from somewhere else at
  runtime, functions the description does not mention;
- a provider whose only job is to load other providers or third-party source lists.

Allowed, at the user's own risk: a small audience, an unofficial API used with the user's own
account, reading a service's web pages, free public content in the quality the service offers
without a subscription, and code that would not pass review for the server.

The rules get their own page next to the usage policy, with how to report a provider and what
happens to a blocked one.

### 6. Server: installing and loading

- **Switch.** A switch on the Catalog page to also show the community catalog. It is off by
  default, admin only, and switching it on asks for an explicit confirmation with the own-risk
  text. A second setting updates everything in it automatically, off by default. Switching the
  first one off hides the community catalog and stops new installs; installed community
  providers keep working until the admin removes them.
- **Index.** While the switch is on or a community provider is installed, the server fetches
  the index of its server line at start and once a day, and keeps the last copy. It picks the
  file from its own version: the nightly, the beta, or the stable minor line.
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
  server line that copy is of the previous line, and installed versions load unless it blocks
  them. A domain that exists built-in is refused. Safe mode loads no community provider.
- **Development mode.** Only in development mode the server also loads providers from a local
  folder given at start, and the app shows that it runs in development mode.
- **Daily check.** A blocked provider is unloaded with the reason. An installed version that is
  not listed for this server line is switched to the newest listed one, or not loaded when none
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

- The switch on the Catalog page ([provider-catalog.md](provider-catalog.md), section 2), with
  its confirmation.
- With the switch on, community providers appear in the Catalog, with the Community badge, the
  maintainer, a link to the provider's own documentation and "Get help" pointing at the
  maintainer's support URL or repository.
- The Catalog shows which installed community providers have an update and which community
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
  fixtures an out-of-tree test needs ship inside the server package.
- **Running it.** In the development container the server starts in development mode with the
  repository as the local provider folder. No local Python, ffmpeg or operating system
  requirement beyond Docker.
- **On Home Assistant.** The Dev app gets a `provider_repo` option with the syntax of the
  existing `server_repo`, so an author or a tester runs a provider from a repository, branch or
  pull request.
- **On Docker.** A new dev image on top of the nightly image
  (`ghcr.io/music-assistant/server:dev`) takes the server, frontend and provider references from
  the environment. The Dev app and the dev image share one reference parser, kept in the server
  repository.
- **CI.** A reusable workflow, owned by the registry, runs the manifest, dependency and load
  checks of section 3 on the author's pull requests. The red-line reading happens only in the
  registry.
- **Release.** Publish a GitHub release.
- **Docs.** A page in the developer documentation: the two places and what each asks, the red
  lines, the template, testing on Home Assistant and on Docker, and what the nightly test
  reports.

### 9. Moving providers out of the server

- A provider leaves the server in a normal release: its folder is removed and its domain is
  added to the list of moved providers. CI refuses that change until the index of that server
  line lists the provider.
- Providers that core imports stay. Providers that import each other move together.
- **First:** the providers whose maintainer already develops and releases them from own
  repositories, in agreement with that maintainer.
- **Providers without an active maintainer.** The server repository asks for a maintainer who
  follows up. Where a provider has had no active maintainer for a long time and the team does
  the fixes, the team may move it; a year without codeowner activity is the working measure. It
  is announced first, in the app and to the codeowner, so that someone can take it over. When
  nobody does, it moves anyway, into one repository in the Music Assistant organisation that
  parks every provider without a maintainer, and it is listed as looking for a maintainer.
  This is code that shipped with the server before, so hosting it is nothing new. Whoever takes
  a provider over moves it to an own repository. A parked provider that stops loading on a
  server line and that nobody repairs leaves the index of that line.
- Issues about a community provider that arrive in the support repository are pointed to the
  maintainer's support URL.

### 10. Contributing to the server

- The READMEs and `DEVELOPMENT.md` state what a built-in provider asks: demand that can be
  shown, a documentation pull request, the code standards with tests, and a named maintainer who
  follows up on issues. They also say that a provider without an active maintainer may move to
  the community catalog.
- A pull request labelled `new-provider` gets one automatic comment with the same text. Once
  the community catalog accepts admissions, the comment also points to it as the other route.

### 11. What a community provider can show in the app

- **Menu actions.** A provider declares actions for a media type (label, icon, the command to
  call). Every client renders them in the context menu and the play menu.
- **Forms.** A provider describes a form with the config entry fields the app already draws.
- **An own page.** A provider can already serve its own HTML on a route of the webserver. The
  app can link to it or show it in a frame.
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

**A pilot.** The design is built once. The first providers and the open pull requests are the
first real use.

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
6. Frontend (section 7), on the Catalog page, which is built first
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
- Switching the community catalog off leaves installed community providers working and
  updatable, and offers no new installs.
- Installing a version that the index of the server's line does not list is refused. A download
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
- A release that imports on the stable line and fails on the nightly appears in the stable
  index only, and its maintainer gets one issue with the output.
- Upgrading a server that uses a moved provider keeps the provider working, with its settings
  and library items, and shows the notice once.
- A settings entry of a removed community provider is shown as unavailable and can be removed.
- A new-provider pull request on the server gets exactly one comment with the requirements.
- A repository made from the template passes the reusable workflow, runs in the development
  container, and loads in the Dev app through `provider_repo`.
