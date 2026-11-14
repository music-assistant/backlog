# Community catalog: how the server installs and loads community providers

The implementers' spec for the pull requests of music-assistant/backlog#437. The plan
[community-catalog.md](community-catalog.md) is the decision record and wins where they disagree.

Scope: section 6 of that plan plus what it needs from sections 1, 2 and 9. All `file:line`
references are `music-assistant/server` at `origin/dev` `ae091818e` (2026-11-13), unless another repository is named.
"Community location" is the folder that becomes the second location of the `music_assistant.providers` package.

## 1. Verified starting point

- **Discovery and manifests.** One folder: `PROVIDERS_PATH` (`mass.py:128`). `__load_provider_manifests`
  (`mass.py:1603-1657`) lists it once at start (`mass.py:295`, before storage and the core controllers), parses
  `manifest.json`, sets `tier = ProviderTier.CORE` (`mass.py:1618`), reads icons (`1620-1623`) and detects
  `setup_flow.py` by presence (`1626-1628`). `_`-prefixed folders load only in dev mode (`1651`). Nothing registers a
  manifest after start; `_provider_manifests` is a plain dict (`mass.py:258`). `ProviderManifest` has no version field
  (models `provider.py:17-67`).
- **Import.** `load_provider_module(domain, requirements)` (`helpers/util.py:1522-1557`) imports
  `music_assistant.providers.<domain>` in a single-worker executor (`util.py:1496-1519`). Only two call sites import
  provider code: `mass.py:1495` and the setup flow (`controllers/config/flows.py:690-693`). Both look the manifest up
  first. `music_assistant/providers/__init__.py` is a one-line regular package.
- Providers use both styles: 1012 relative imports and 433 absolute `music_assistant.providers.<domain>…` imports.
  Shared code is imported absolutely (`music_assistant.helpers` 446×, `.models` 517×, …). Cross-provider imports exist
  without `depends_on` (`telmore`/`yousee` → `music247e`, `filesystem_onedrive` → `filesystem_cloud`), so `depends_on`
  does not describe the import graph.
- **Requirements.** Per pinned requirement `importlib.metadata` is compared and `uv pip install --no-cache <req>` runs
  into the active environment (`util.py:1446-1453`, `1530-1546`); on `ImportError` everything is installed again
  (`1551-1554`). No constraints, no target. `uv` is a runtime dependency (`pyproject.toml:42`, `uv>=0.8.0`):
  `/app/venv/bin/uv`.
- **Load, unload, reload.** `_load_providers` (`mass.py:1383-1454`) loads enabled configs with bounded concurrency;
  `load_provider` (`1073-1182`) records `last_error` and retries only `MusicAssistantError`s (`1138-1145`);
  `load_provider_config` (`1001-1071`) raises; `_load_provider` (`1456-1521`) unloads first, imports, runs `setup`.
  `unload_provider` (`1184-1232`) cascades over `depends_on`. Reload is `config/providers/reload`
  (`controllers/config/providers.py:684-700`). Nothing touches `sys.modules`.
- The mDNS browser is built once from all manifests (`controllers/discovery/controller.py:269-285`). There is no
  server restart command (grep for `restart` in `mass.py`, `__main__.py`, webserver).
- **Safe mode, dev mode.** Safe mode: `--safe-mode`, HA option or `MASS_SAFE_MODE` (`__main__.py:75-79`, `240-242`);
  it skips `_load_providers` (`mass.py:393-394`). Dev mode: `PYTHONDEVMODE=1` or `.venv` next to the package
  (`mass.py:268-271`); only used at `mass.py:1651`. `ServerInfoMessage` carries neither.
- **Config entries without code.** `get_provider_configs` drops a config whose domain has no manifest
  (`providers.py:113-124`); `get_provider_config` raises `KeyError` (`147-154`); `remove_provider_config` raises
  because `get_provider_manifest` does (`providers.py:492`, `mass.py:544-550`). The cleanup itself only needs the
  stored `type` (`providers.py:499-517`).
- The admin is told about a provider problem through `ProviderConfig.last_error` and the derived `ProviderStatus`
  (`controllers/config/helpers.py:55-71`, `providers.py:542-553`).
- **Texts.** Runtime reads only `translations/en.json` and, lazily, `translations/<lang>.json`
  (`controllers/translations/__init__.py:73-83`, `115-137`); keys are flat `provider.<domain>.…`. `strings.json` is
  compiled at build time (`scripts/build_translations.py:79-99`, `141-148`).
- **Data, version, API.** Data dir: `--data-dir`, default `~/.musicassistant` (`__main__.py:43-60`); cache dir
  defaults to `<data>/.cache`. Settings are `<data>/settings.json` (`controllers/config/controller.py:69`). The cache
  dir already holds re-downloadable heavy data (torch and Hugging Face, `util.py:324-333`).
- Version: `MASS_VERSION` from package metadata, `0.0.0` from source (`constants.py:37-45`). Formats: `X.Y.Z`,
  `X.Y.ZrcN`, `X.Y.ZbN`, `X.Y.Z.devN` (`.github/workflows/RELEASE_WORKFLOW_GUIDE.md:11-14`). Channel mapping exists
  once, private: `DashboardController._frontend_channel` (`controllers/dashboard/controller.py:419-426`).
- Commands: `@api_command(name, required_scope=…)` (`helpers/api.py:184-212`), collected from a fixed list of objects
  (`mass.py:1284-1324`). `Scope.SYSTEM_MANAGE` and `CONFIG_CORE_WRITE` can only be held by the admin role
  (`controllers/webserver/helpers/auth_middleware.py:80-90`).
- Core settings are `ConfigEntry`s of a `CoreController` (`models/core_controller.py:59-66`), stored under
  `core/<domain>/values`, saved with `config/core/save`, which calls `update_config`
  (`controllers/config/core.py:171-217`). The block is only written on save.
- API result models can be server-local (`controllers/storage/models.py:49`). The frontend types are hand-written
  (frontend `src/plugins/api/interfaces.ts`); it fetches manifests once per connect (frontend
  `src/plugins/api/index.ts:4084`).
- Scheduled tasks can be disabled by an admin (`controllers/tasks/controller.py:195-241`). Precedent for download,
  sha256, swap and rollback: `providers/spotify_connect/soloist/runtime.py:543-696`.
- **Image and Home Assistant app.** venv `/app/venv` (`Dockerfile:61`, `125-126`), `VOLUME /data` (`157`),
  entrypoint `--data-dir /data --cache-dir /data/.cache` (`172`). The compiler exists only in the builder stage (`Dockerfile:53-56`).
  `requirements_all.txt` is copied into the builder only (`Dockerfile:42`), so the running server has no pin file. The
  image installs with the PyTorch CPU index and `--index-strategy unsafe-best-match` (`Dockerfile:71-74`,
  `requirements_all.txt:3`).
- HA app (`home-assistant-addon` `music_assistant/config.yaml`): same image, `tmpfs: true` (so `/tmp` is RAM),
  `backup_exclude` contains `.cache/*`. Its AppArmor profile has a blanket `file,` rule.

**Not verified**: native wheels loading from `/data` on a real HA app; the uv version in the published image; the
spikes ran on macOS arm64.

## 2. Design

- New package `music_assistant/controllers/community_catalog/`. `controller.py`:
  `CommunityCatalogController(CoreController)`, domain `community_catalog` (settings, API, reconcile, timer);
  `index.py`: `server_line()`, `IndexClient`; `installer.py`: download, extract, `content_hash()`, swap;
  `dependencies.py`: `CommunityDependencies`; `gate.py`: `evaluate()`, the pure decision function; `models.py`,
  `constants.py`, `strings.json`, `README.md`.

```
# on disk, created on first use, never at upgrade
<data>/community_providers/        second location of music_assistant.providers
  <domain>/                        live code
  .previous/<domain>/              one earlier version
  .staging/<domain>/               download target (same filesystem; /tmp is RAM on HAOS)
  state.json  index.json
<cache>/community_deps/            uv target folder + requirements.lock
```

### 2.1 The two settings

- Two `ConfigEntry`s on the controller: `catalog_enabled` (S6) and `auto_update` (S8, together with what it does),
  both `BOOLEAN`, default `False`, `hidden=True`, `auto_update` with `depends_on="catalog_enabled"`. The key cannot
  be `enabled`: `Config.update` in the models takes `enabled` and `name` for the fields of a provider config. Label
  and description live in the controller's `strings.json`; the description of `catalog_enabled` is the own-risk
  text, so the frontend dialog shows a server-translated text. Saved through the existing `config/core/save`
  (`CONFIG_CORE_WRITE`, admin only).
- The controller is not added to `CONFIGURABLE_CORE_CONTROLLERS`. `config/core/get`, `get_entries` and `save` look a
  controller up by its domain and work without it; the membership only puts a controller in the `config/core` list
  and among the manifests, and the app must not show a "Community catalog" item in its core settings.
- `update_config`: the switch turning on triggers an index fetch. Turning it off unloads nothing; the list then
  returns installed providers only and `install` of a new domain is refused.
- `_needs_index() -> bool`: the switch, or `state.installed`, or a config entry of a moved domain. False means no
  request and no timer.
- Rejected: a top-level settings key with own commands (a second way to store settings). Fails safely: an absent block
  reads as off.

### 2.2 Index client

- `server_line(version) -> str`: `0.0.0` or `.dev` → `nightly`; `b` or `rc` → `beta`; else `<major>.<minor>`. The
  channel rule moves from `DashboardController._frontend_channel` to `helpers/build_info.py::release_channel(version)`
  and both use it.
- `INDEX_URL: Final = "https://music-assistant.github.io/community-catalog/v1/{line}.json"`, a module constant. No
  setting, no environment variable, also not in dev mode. The host is the GitHub Pages site of the registry
  repository; the registry step confirms or replaces it before the app shows the switch.
- `IndexClient.fetch() -> bool` (changed): `mass.http_session`, `If-None-Match` from `state.index_etag`, 10 s timeout.
  200 with `schema == 1` and matching `server_line` is written atomically to `index.json`; 304 keeps the copy;
  anything else (404 for a line without index, timeout, parse error, unknown schema) keeps the copy and logs.
- Timing: once at start in a background task, then `mass.call_later(24 h + random 0-60 min)`, private timer. Rejected:
  a scheduled task (an admin can disable it, which would disable the kill switch).
- Kept copy: `index.json`, which carries its own `server_line`; `state.json` holds the ETag. A copy of another line
  is used for its blocks only (2.6) and lists nothing to install. No copy at all: no community provider loads (fail
  closed).
- Models stay server-local in `models.py` (precedent `controllers/storage/models.py`); only two enum values and
  `dev_mode` go to `music-assistant-models` (section 4, M1 and M2). The server never compares version strings: `latest` is
  the registry's call. A withdrawn entry keeps loading what is installed and offers no install or update.

```python
class IndexVersion: commit: str; folder: str; content_hash: str; requirements: list[str]
                    released: str  # when the version was released
class IndexProvider:
    status: IndexStatus                # listed | withdrawn | blocked
    reason: str | None; reason_url: str | None
    repository: str; maintainers: list[str]; icon_svg: str | None
    added: str                         # when the provider entered the catalog
    manifest: dict[str, Any]           # ProviderManifest snapshot of `latest`
    latest: str | None                 # newest version that passed for this line
    versions: dict[str, IndexVersion]
    blocked_versions: dict[str, str]   # version -> reason
class CatalogIndex: schema: int; server_line: str; generated: str; providers: dict[str, IndexProvider]
```

### 2.3 Install: download, content hash, swap, previous

```python
# installer.py, all blocking work in threads
async def stage(mass, domain: str, entry: IndexProvider, version: str) -> Path   # raises InstallError
def activate(domain: str) -> None          # .staging -> live, live -> .previous
def restore_previous(domain: str) -> None  # .previous -> live
def remove(domain: str) -> None
def content_hash(folder: Path) -> str
```

- `stage`: stream `https://codeload.github.com/{repository}/tar.gz/{commit}` to `.staging/`; open with `tarfile`; drop
  the first path component; take only members under `folder/`; a member in that subtree that is not a regular file or
  directory fails the install; only the executable bit is kept. Then: `manifest.json` parses, its `domain` equals the
  index domain, the domain is not built-in, every requirement is a plain name with specifier (no URL, no option line),
  and `content_hash(staged) == IndexVersion.content_hash`. Any failure deletes the staged folder and leaves the
  installed version untouched.
- **Content hash** (the definition the registry must use): for every regular file below the provider folder, except
  anything inside a `__pycache__` directory and `*.pyc`, build the line `<sha256 hex of the file bytes><two
  spaces><POSIX path relative to the folder>\n`; sort the lines by path as UTF-8 bytes; the hash is the sha256 hex of
  the concatenation. Modes, timestamps and empty directories do not count. It equals `find . -type f … | LC_ALL=C sort
  | xargs sha256sum | sha256sum` and must be computed from the GitHub tarball of the commit, not from a checkout.
- `activate`: delete `.previous/<domain>`, rename live to `.previous`, rename staged to live, then write `state.json`
  (temp file and rename). A crash in between leaves a folder whose hash differs from the state, which the gate repairs
  by reinstalling (2.6). Rejected: `git clone` (needs git in the image), hashing the archive (GitHub may regenerate
  it).

```python
class InstalledProvider:               # one per domain in state.json
    version: str; commit: str; content_hash: str
    verified: bool = False             # this version loaded successfully once
    previous_version: str | None = None; moved_notice: bool = False
    pending_version: str | None = None # staged, activates at the next start (2.7)
    failed_version: str | None = None  # rolled back from; not retried
```

### 2.4 Dependencies

```python
# dependencies.py, class CommunityDependencies(mass)
async def resolve(self, requirements: Mapping[str, Sequence[str]]) -> ResolvedDependencies
async def touches_imported(self, pins: Sequence[str]) -> bool
async def sync(self, resolved: ResolvedDependencies) -> None
def activate(self) -> None                       # sys.path.append(<cache>/community_deps), once
class ResolvedDependencies                       # pins: tuple[str, ...]; inputs_hash: str
class DependencyConflict(MusicAssistantError)    # .domain is refused; .culprit: "server", another domain,
                                                 # or the domain itself when it cannot be installed at all
class DependenciesUnavailable(MusicAssistantError)  # uv or the network failed
```

- Constraints at runtime: `importlib.metadata.distributions(path=[purelib, platlib])` of the running interpreter,
  written as `name==version` exactly as installed, local version part (`+cpu`) included. Listing by path keeps the
  community folder itself out. This is what "the server's own pins" means in a running image, where no pin file
  exists.
- uv resolves against the indexes the image is built with: PyPI plus the PyTorch CPU index, with
  `--index-strategy unsafe-best-match`. Against PyPI alone the server's `torch` would be resolved with the
  requirements of the CUDA build, and a provider that needs torch would get 19 `nvidia-*`, `cuda-*` and `triton`
  packages installed next to it (spike a10). A test keeps the two in step with `requirements_all.txt`.
- `resolve` first refuses every requirement that is not a package name with optional extras and version
  specifiers: a URL or an option line would otherwise reach uv. It then runs one pass over the union of all loadable
  community (and dev) requirements: `uv pip compile reqs.in --constraint server.txt -o <output>` with `--python
  sys.executable --no-config --no-cache --only-binary :all: --no-header --no-annotate`. The output file is filled
  with the current pins first, so they stay unless a requirement forces a change (spike a6). From the output the
  server's own packages are dropped by name; what is left is what gets installed. The versions chosen are
  therefore exactly the ones proven against the server.
- Not `--excludes`: with the server's packages excluded uv ignores what other packages require of them, and picks
  a newest version that needs a newer server package than the server has (spike a9).
- `resolve` returns the installed set without calling uv when the hash of its inputs equals the one in the lock,
  and `sync` skips uv when the pins equal the lock: `uv pip sync` contacts the index even when nothing changes.
- Exit code 1 is a conflict, anything else non-zero is `DependenciesUnavailable` (spike a4/a5). On a conflict while
  adding provider X the culprit is found by resolving X alone (fails: `server`) and then X with each other provider
  (first failure names it); uv's cause line goes into the message.
- `sync`: `uv pip sync requirements.lock --target <cache>/community_deps` with the same flags. It installs, replaces
  and removes until the folder equals the lock (spike a7), so install, update, remove and a server upgrade are one
  code path.
- The lock also records the Python version and platform it was installed for (`cpython-314-linux-x86_64`). When
  that differs, `resolve` runs again and `sync` passes `--reinstall`: a wheel with native code only fits the
  interpreter it was installed for, and uv does not replace a package whose version is unchanged.
- `sys.path`: the folder is appended last with `sys.path.append`, not `site.addsitedir` (no `.pth` execution). A
  server package always wins an import (spike a3), and it is not even installed there.
- When: at start, before any community provider loads and with nothing imported from the folder yet, the set is
  resolved again whenever the hash of (constraints + requirements) differs from the one stored next to the lock. That
  covers a server upgrade. At runtime see 2.7; a removal leaves the folder alone until the next start.
- Offline: `DependenciesUnavailable` at start keeps the folder as it is and loads; a provider that then fails to
  import shows that error, and the sync is retried at every daily check. Offline during an install refuses it.
- uv runs as `sys.executable -m uv`, the uv that is installed with the server. No newer uv is needed.
- Rejected: `uv pip install --target` on the requirements directly (copies every package the server already has, spike
  a1; with torch it would pull the CUDA wheels from PyPI). Fails safely: a refused set changes nothing on disk; the
  lock file is written after a successful sync.

### 2.5 Second location and load path

- `MusicAssistant.add_provider_location(path)`: `music_assistant.providers.__path__.append(path)`. Appending keeps the
  built-in folder first, so a built-in domain can never be shadowed (spike b7). Relative imports, absolute
  `music_assistant.providers.<domain>.x` imports and imports of built-in providers work unchanged (spike b2): the
  module name is what it would be built-in. The installed folder is named after the domain, whatever the repository
  calls it.
- `MusicAssistant.register_provider_manifest(provider_path: str, tier: ProviderTier) -> ProviderManifest` and
  `unregister_provider_manifest(domain: str) -> None`. The body of the inner `load_provider_manifest`
  (`mass.py:1606-1643`) becomes the shared private parser. For `ProviderTier.COMMUNITY` it forces `tier`,
  `builtin=False`, `allow_disable=True`, requires the folder name to equal the domain, and raises when the domain is
  already registered (built-in provider or core controller). Unregister also drops the icons and cancels
  `load_provider_<instance>` timers.
- **A community manifest is registered only while its provider may load.** Both import paths start from the manifest,
  so "not registered" is the whole gate: no second check to forget.
- `load_provider_module(manifest: ProviderManifest)` replaces `(domain, requirements)`; for community tier it only
  imports. The existing install path can then never write a community requirement into the server environment. Two
  callers change (`mass.py:1495`, `flows.py:690`).
- `purge_provider_modules(domain: str) -> None` in `helpers/util.py`: delete `music_assistant.providers.<domain>` and
  its submodules from `sys.modules` and the attribute on the package, `importlib.invalidate_caches()` (spike b5/b6).
- `DiscoveryController.refresh_mdns_browser()` (public wrapper of `_setup_mdns_browser`), called when a registered or
  unregistered manifest has `mdns_discovery`.
- The controller loads community instances with `mass.load_provider(instance_id, allow_retry=True)` under
  `TaskManager(mass, PROVIDER_LOAD_CONCURRENCY)`; `_load_providers` skips them, their manifest is not there yet.
  Rejected: a custom import finder (more machinery than one `append`); a separate package name (breaks absolute
  imports of moved providers).

### 2.6 Load gate, start and daily check

One pure function decides, one method acts, at start and after every fetch:

```python
def evaluate(installed: InstalledProvider, folder_hash: str | None, entry: IndexProvider | None,
             *, index_is_own_line: bool, auto_update: bool) -> Decision
```

| First matching rule | Decision |
|---|---|
| no index copy at all | `UNAVAILABLE` (catalog not reachable) |
| entry `blocked`, or version in `blocked_versions` | `BLOCKED(reason, url)` |
| index copy is of another line | `LOAD` if `folder_hash == installed.content_hash`, else `UNAVAILABLE` |
| entry missing | `UNAVAILABLE` (no longer listed) |
| version not in `versions`, or its hash differs | `SWITCH(latest)` if `latest` exists and is not `failed_version`, else `UNAVAILABLE` (not available for this server version) |
| `folder_hash != installed.content_hash` | `REINSTALL(version)` |
| `latest != version`, not `failed_version`, entry `listed` | `UPDATE(latest)` if `auto_update`, else `LOAD` with update offered |
| otherwise | `LOAD` |

The folder hash is computed at every start (a few ms per provider), so a folder copied in or edited by hand never
loads; it is replaced by the listed version.

An automatic `UPDATE` at the daily check is carried out only while no player is playing. Otherwise the server tries
again every hour until it is done. An update the admin asks for is carried out at once.

`CommunityCatalogController.start()` is called from `MusicAssistant.start()` after `_load_providers()` inside the
existing `if not self.safe_mode` block, and runs as a background task:

1. add the location and the deps folder to the paths when they exist; register dev-folder providers (2.10);
2. when `_needs_index()`: fetch (10 s), fall back to the copy;
3. activate `pending_version`s (2.7); install moved domains (2.8);
4. `evaluate` every installed provider; perform `SWITCH`, `REINSTALL`, `UPDATE`;
5. resolve and sync dependencies for everything that may load; a provider named as culprit becomes `UNAVAILABLE`
   (requirements clash with this server version) and the rest is resolved again;
6. register manifests and texts, refresh mDNS, load instances; signal `COMMUNITY_CATALOG_UPDATED`;
7. schedule the daily check, which is steps 2 and 4-6 against the running state.

All mutating work (reconcile, install, update, remove) runs under one `asyncio.Lock`. In safe mode `start()` is never
called: nothing is added to `__path__` or `sys.path`, nothing is fetched, no community manifest exists, and the
commands raise `ActionUnavailable`.

What the admin is told, all through existing mechanisms:
- `BLOCKED` / `UNAVAILABLE`: instances are unloaded, modules purged, the manifest unregistered. The config entries
  stay and are served as entries without a provider (2.9) with the reason in `last_error`
  (`ProviderError(error_code=ProviderUnavailableError.error_code, translation_key=…, translation_args=[reason])`).
  Files stay on disk; settings and library items are untouched.
- A failed load of an unverified version: 2.7. A warning is logged in every case, and the list command carries
  `state`, `reason`, `reason_url`.

### 2.7 Update in a running server, rollback, when a restart is needed

`_switch(domain, version)` serves install, update, switch, reinstall and rollback:

1. `stage`; `resolve` with the staged manifest's requirements (conflict: refuse, name the culprit).
2. If `touches_imported(pins)`, i.e. the new lock replaces or removes a distribution in the deps folder one of whose
   top-level modules is in `sys.modules`: keep the staged folder, set `pending_version`, report `restart_required`.
   The running version keeps running; step 3 of `start()` activates it next time. A first install never hits this
   unless it moves a shared pin.
3. Otherwise live: unload the domain and every community provider whose loaded modules reference it (scan of their
   module globals for objects whose `__module__` is under the domain; transitive), purge, `activate`, `sync`,
   register, import, load. Unrelated providers are not touched.
4. Rollback: while `verified` is false, an import error or a non-`MusicAssistantError` from load (the server's
   existing line between a bug and a handled condition, `mass.py:1138-1145`) calls `restore_previous`, sets
   `failed_version`, and loads the previous version when `evaluate` allows it; otherwise the provider is `UNAVAILABLE`
   with "version X failed to load". Auth and network errors do not roll back. `verified` is set on the first
   successful load (or import, with no enabled instance).

- Rejected: always restart (no restart command exists and a first install would need one); purging third-party modules
  (unsafe for C extensions); reloading every community provider on each update (stops unrelated players). Fails
  safely: every step before `activate` leaves the installed version in place.

### 2.8 Moved providers

- `MOVED_PROVIDER_DOMAINS: Final[frozenset[str]] = frozenset()` in `constants.py`, filled by step 8.
- Start step 3: a config entry of a moved domain that is not installed makes `_needs_index()` true and installs
  `latest`, with the switch off; the list still returns installed providers only. Not listed or offline: the entry is
  shown as unavailable (2.9) and retried at each daily check.
- `moved_notice=True` on that install; the list returns `notice="moved"` until `community_catalog/dismiss_notice`.
  Rejected: a migration at upgrade (needs network inside a settings migration).

### 2.9 Entries without a provider

- `controllers/config/providers.py`, `get_provider_configs(..., include_unavailable: bool = False)`: with the flag, a
  config whose domain has no manifest is returned as `ProviderConfig.parse([], raw)` with `status =
  ProviderStatus.UNAVAILABLE` and `last_error` from `mass.community_catalog.unavailable_reason(domain)` or the generic
  "This provider is no longer available". Stamped on read, never stored. Internal callers and existing clients keep
  today's result.
- `get_provider_config` returns the same object instead of raising.
- `remove_provider_config` looks the manifest up with `.get`: refuse when built-in, otherwise the unchanged cleanup
  (`providers.py:496-517`). Rejected: always including them (internal callers look the manifest up, e.g.
  `mass.py:1043`).

### 2.10 Local folder in development mode

- `--dev-providers-dir PATH` in `__main__.py`, passed to `MusicAssistant`; honoured only when `mass.dev_mode`,
  otherwise a warning. Every subfolder with a `manifest.json` whose folder name equals its domain is registered as
  community tier without index or hash; a built-in domain is refused. Their requirements join the same dependency
  resolve. `ServerInfoMessage.dev_mode` tells the app. Rejected: an environment variable for the index URL (a custom
  repository through the back door).

### 2.11 Texts

```python
# TranslationController (controllers/translations/__init__.py)
async def load_provider_strings(self, domain: str, provider_path: str) -> None
def unload_provider_strings(self, domain: str) -> None
```

- `strings.json` is flattened under `provider.<domain>.` into `_source`; `[%key:…%]` reference values are skipped, the
  owner → common fallback resolves them (`_candidate_keys`). Optional `translations/<lang>.json` files have the shape
  of `strings.json`; they are merged into a loaded locale at once and otherwise when `ensure_locale_loaded` loads it.
  The forced prefix means a community provider cannot change `common.` or another provider's texts. Called from
  register and unregister; a broken file logs a warning and loads nothing. Rejected: compiling them into
  `translations/*.json` (package files, not writable data).

### 2.12 Admin API and events

All on the controller (added to the list in `mass._register_api_commands`), `Scope.SYSTEM_MANAGE`:

| Command | Arguments | Result |
|---|---|---|
| `community_catalog/providers` | - | `list[CommunityProvider]`: installed ones always, the rest of the index when switched on |
| `community_catalog/install` | `domain` | `CommunityProvider`; newest listed version; refused when off and not installed |
| `community_catalog/update` | `domain: str \| None` | `list[CommunityProvider]`; `None` updates all |
| `community_catalog/remove` | `domain` | `None`; unloads, deletes code, keeps config entries |
| `community_catalog/refresh` | - | `list[CommunityProvider]` after a fetch now |
| `community_catalog/dismiss_notice` | `domain` | `None` |

```python
class CommunityProvider(DataClassDictMixin):
    domain: str; repository: str; maintainers: list[str]
    icon: str | None                # a data URI, like providers/icon returns
    manifest: ProviderManifest      # installed folder, else the index snapshot
    state: CommunityProviderState   # available | installed | update_available | restart_required | blocked
                                    # | unavailable | withdrawn
    installed_version: str | None; latest_version: str | None; pending_version: str | None
    added: str | None               # when it entered the catalog
    released: str | None            # when latest_version was released
    reason: str | None; reason_url: str | None; notice: str | None
```

- Install does not create a config entry: the manifest appears in `providers/manifests` and the admin adds the
  provider through the existing flow.
- An index entry is listed as available only with status `listed` and a version record for its `latest`.
- The icon of the index (`icon_svg`, SVG text) is handed to clients as a data URI only. A client shows it as an
  image; markup from a community provider never reaches a page.
- `EventType.COMMUNITY_CATALOG_UPDATED` (no data) after anything that changes the list or the registered manifests;
  the frontend refetches the list and `providers/manifests`. Loading and unloading keep sending `PROVIDERS_UPDATED`.

## 3. Stored data

| What | Where | Existing install on upgrade |
|---|---|---|
| `catalog_enabled`, `auto_update` | `settings.json` → `core/community_catalog/values` | absent, reads as off; written on first save |
| live code, `.previous/`, `.staging/` | `<data>/community_providers/` | not created until the first fetch or install |
| `state.json` (`index_etag`, `installed`) | same folder | absent |
| `index.json` (last fetched index) | same folder | absent |
| dependencies and `requirements.lock` | `<cache>/community_deps/` (`/data/.cache/…` in Docker and HAOS) | absent; rebuilt from the lock inputs when missing |

- No database change, no change to existing keys, **no migration**. Config entries of community providers are ordinary
  `providers/<instance_id>` entries.
- State lives next to the code it describes, not in `settings.json`: deleting the folder resets both, and no settings
  migration ever has to know it.
- HA backups include `community_providers/` (small) and skip `.cache/*`, so a restore on another architecture rebuilds
  the wheels.
- `state.json` starts with `"schema": 1`; an unknown schema reads as empty state (nothing loads, nothing is deleted).

## 4. Sequence of pull requests

Each PR targets `dev`, is `maintenance` until the frontend makes it visible, and changes nothing for a server with the
switch off.

| # | Goal | Files | Tests | Needs first |
|---|---|---|---|---|
| M1 | `ProviderStatus.UNAVAILABLE`, `EventType.COMMUNITY_CATALOG_UPDATED` | models `enums.py` | - | - |
| M2 | `ServerInfoMessage.dev_mode`, with the `API_SCHEMA_VERSION` bump in S5 | models `api.py` | - | - |
| S1 | Entries without a provider are listed on request and removable | `controllers/config/providers.py`, `helpers.py`, `music_assistant/strings.json` | `tests/controllers/config/test_unavailable_providers.py`: default result unchanged; flag returns status and reason; get does not raise; remove runs the music/player cleanup; built-in still refused | M1 |
| S2 | Second location, manifest (un)register, import-only load for community tier, module purge, mDNS refresh | `mass.py`, `helpers/util.py`, `controllers/config/flows.py`, `controllers/discovery/controller.py` | `tests/core/test_provider_locations.py`: relative and absolute imports from the second folder; built-in and core domains refused; `tier`/`builtin` forced; purge then import gives the new code; folder name ≠ domain refused. Adjust `tests/helpers/test_util.py`, `tests/controllers/config/test_setup_flows.py`, `tests/test_provider_manifests.py` | - |
| S3 | Provider texts merged at load | `controllers/translations/__init__.py` | `tests/controllers/translations/test_provider_strings.py`: prefix forced; reference values fall back to common; locale file merged before and after the locale is loaded; unload removes | - |
| S4 | Dependency resolve and sync | `controllers/community_catalog/dependencies.py` | `tests/controllers/community_catalog/test_dependencies.py`, hermetic: wheels built in the test, a module-level `_UV_INDEX_ARGS` patched to `--no-index --find-links`: server package excluded; clash with server names `server`; clash between two providers names the other; no wheel refused; a newer version that does not fit the server is passed over; a URL or option requirement refused; lock kept stable; `touches_imported`; exit code ≠ 1 is unavailable | - |
| S5 | Controller skeleton and the dev-mode local folder | `controllers/community_catalog/{__init__,controller,constants}.py`, `mass.py`, `__main__.py`, `DEVELOPMENT.md` | `tests/controllers/community_catalog/test_dev_folder.py`: loads as community in dev mode; ignored outside dev mode; not in safe mode; `dev_mode` in server info | M2, S2-S4 |
| S6 | The switch, index client, kept copy, list and refresh commands | `controller.py`, `index.py`, `models.py`, `strings.json`, `constants.py`, `helpers/build_info.py`, `controllers/dashboard/controller.py`, `translations/en.json` | `test_index.py`: line mapping for `0.0.0`, `.dev`, `b`, `rc`, `2.11.3`; ETag 304; 404/timeout/bad schema keep the copy; no request when off and nothing installed; `test_settings.py` | S5 |
| S7 | Install, remove, gate at start, blocked at the daily check | `installer.py`, `gate.py`, `controller.py`, `README.md` | `test_installer.py`: only the folder is taken; symlink refused; hash mismatch leaves the installed version; hash equals a coreutils-computed fixture; crash between renames is repaired. `test_gate.py`: one case per table row. `test_controller.py`: blocked provider unloads, entry stays with reason; hand-copied folder does not load | S1, S6 |
| S8 | Update, the `auto_update` setting, switch of unlisted versions, rollback, pending | `controller.py`, `installer.py` | `test_update.py`: live update reloads importers only; imported dependency change sets `pending_version` and applies at start; unverified version that raises `TypeError` rolls back and sets `failed_version`; `LoginFailed` does not; unlisted switches or becomes unavailable | S7 |
| S9 | Moved providers and the notice | `constants.py`, `controller.py` | `test_moved.py`: installs with the switch off; list stays limited to installed; notice dismissed once | S8 |

Before the registry exists: everything through S9 is built against fixtures. Tests serve an index JSON and tarballs
built with `tarfile` from an `aiohttp` test server and patch the two URL constants (`INDEX_URL`, `ARCHIVE_URL`). A
manual end-to-end run uses a launcher script outside the package that patches the same constants before calling
`main()`; the shipped server has no way to change them. The frontend (#438) can start after S1 and S6.

What this step needs from other steps:
- **Registry (#435)**: the final index URL with a version segment; the index fields of 2.2 (`schema`, `server_line`,
  per provider `status`/`reason`/`reason_url`/`repository`/`maintainers`/`added`/
  `manifest`/`icon_svg`/`latest`/`versions{commit, folder, content_hash, requirements, released}`/
  `blocked_versions`); blocked
  and withdrawn entries keep their version records; the content hash of 2.3 computed from the GitHub tarball;
  requirements limited to names with specifiers; an SVG icon without scripts or references to other files; the
  dependency check run as the one uv pass of 2.4 against the
  image's installed distributions; a hand-written `nightly.json` with one test provider as early as possible.
- **Developer workflow (#439)**: the provider sits in a folder named after its domain; the dev container passes its
  parent as `--dev-providers-dir` with `PYTHONDEVMODE=1`; the docs say a provider never writes into its own folder (it
  would change the hash).
- **Frontend (#438)**: `include_unavailable`, the six commands, the event, `dev_mode`, the confirm dialog on the
  hidden `catalog_enabled` entry (read with `config/core/get_entries`, domain `community_catalog`), and the icon of
  a listed provider shown as an image from its data URI.
- **Moving (#440)**: fills `MOVED_PROVIDER_DOMAINS`.

## 5. Spikes

Scratch venv, Python 3.14.10, macOS arm64, uv 0.13.7; exit codes and behaviour are the same on uv 0.10.6 (the version
in the main checkout's venv). A cold compile of four requirements takes 0.3 s.

**(a) uv with constraints.** `constraints.txt` = `uv pip freeze` of a venv with `aiohttp==3.14.4`, `orjson`,
`mashumaro` (12 pins).

| # | Command (common flags `--python env/bin/python --constraint constraints.txt --only-binary :all: --no-cache`) | Result |
|---|---|---|
| a1 | `uv pip install --target deps1 aiohttp-retry==2.9.1` | exit 0, but installs 10 packages: aiohttp and 8 of its dependencies are copied into the target |
| a2 | `uv pip compile req.in`, then drop server names, `uv pip install --no-deps --target deps3 -r lock` | installs only the 5 new packages |
| a3 | `sys.path.append("deps3")`, import `aiohttp_retry`, `aiohttp` | `aiohttp_retry` from the target, `aiohttp` from the venv |
| a4 | compile with `yarl<1.20`, `aiohttp==3.9.5`, `gql[aiohttp]==3.4.1`, two files pinning `requests` differently, an sdist-only package | exit **1**; `Because you require aiohttp==3.9.5 and aiohttp==3.14.4, …unsatisfiable`; transitive: `gql[aiohttp]==3.4.1 depends on aiohttp>=3.7.1,<3.9.0 and aiohttp==3.14.4`; `…has no usable wheels`. The package is named, the origin (server or which provider) is not |
| a5 | compile with `--default-index http://127.0.0.1:9/simple` | exit **2**, `Request failed after 3 retries` |
| a6 | compile `annotated-types>=0.5` with `-o out.lock` already holding `==0.6.0` | stays `0.6.0` (without the file: `0.8.0`) |
| a7 | `uv pip install --target` of a newer pin; `uv pip sync --target T lock` | old version uninstalled and replaced; sync removes what the lock lacks |
| a8 | compile with `--excludes <server names>` | output is exactly the 5 new packages; but a clash with an excluded package is **not** reported (exit 0) |
| a10 | compile `torch` for Linux x86_64 with constraint `torch==2.13.0` on PyPI alone; then with `torch==2.13.0+cpu`, the PyTorch CPU index and `unsafe-best-match` | PyPI alone: 29 packages, 19 of them `nvidia-*`, `cuda-*` or `triton`. With the CPU index: 10 packages, none of those |
| a9 | server has `srv==1.0`; `foo 1.0` needs `srv>=1`, `foo 2.0` needs `srv>=2`; compile `foo` with the constraint, with and without `--excludes srv` | without: `foo==1.0`. With: `foo==2.0`, which does not fit the server. So `--excludes` is not used |

**(b) Second `__path__` entry.** Package `ma.providers` with a built-in provider; a temp folder appended to
`ma.providers.__path__`; a provider `comm` there using `from .const import …`, `from .sub.deep import …` (which uses
`from ..const`), `from ma.providers.comm.const import …` and `from ma.providers.builtin_one import …`.

| # | Step | Result |
|---|---|---|
| b2 | `importlib.import_module(".comm", "ma.providers")` | imports; `__name__` is `ma.providers.comm`; all four import styles resolve |
| b4 | replace the folder with v2, import again | still v1 |
| b5/b6 | delete `ma.providers.comm*` from `sys.modules`, `delattr` on the package, `invalidate_caches()`, import | v2 everywhere; old module objects still say v1 |
| b7 | a folder named like the built-in provider in the appended location | the built-in one is imported |
| b8 | the same with the location inserted first | the copy wins, so the location must be appended |

**(c) Content hash and archive.** `github.com/<repo>/archive/<sha>.tar.gz` redirects (302) to
`codeload.github.com/<repo>/tar.gz/<sha>`; one top-level folder `<repo>-<sha>/`. The hash of 2.3 computed from the
tarball, from the extracted folder and with `find | sort | shasum` is identical (`eb50b388…` for 36 files of
`music-assistant/models`).

## 6. Decisions (maintainer, 2026-11-13)

1. **Updates are applied live where that is possible.** The provider is reloaded and the new version is active at
   once. Only an update that changes a dependency that is already imported is staged; the app then says "restart
   Music Assistant to finish". No restart command is added.
2. **Automatic updates wait until nothing is playing.** See 2.6. A manual update is carried out at once.
3. **The dependency folder is `<cache>/community_deps`** (`/data/.cache/community_deps` in Docker and the Home
   Assistant app). It survives image updates and stays out of Home Assistant backups; after a restore the wheels are
   fetched again.
4. **The two settings are hidden entries.** The switch on the Catalog, with its confirmation, is the only place to
   turn the community catalog on.
5. **`ServerInfoMessage.dev_mode` is added**, with an `API_SCHEMA_VERSION` bump, in the pull request that fills it.
6. **The registry is `music-assistant/community-catalog`.**
7. **No feature flag.** The pull requests land in `dev` one by one. What keeps them without effect is the product's own
   setting, the community catalog being off until an admin turns it on.

## 7. Risks

- **Stale objects after an in-process replace.** Something outside the provider may hold a class or callback of the
  old module. Contained by reloading the providers that import it, by staging any update that touches an imported
  dependency, and by the fact that a restart always yields a clean state.
- **PyPI or GitHub unreachable.** At start the server keeps the index copy and the dependency folder and loads;
  installs and updates are refused whole; nothing half-installed is ever live.
- **Kill switch latency.** A running server sees a block at its next daily check (up to about 25 h) or start; one that
  can never reach the index keeps its copy. That is the agreed design; with no copy at all nothing loads.
- **Resolver drift.** The server resolves unpinned transitive requirements later than the registry tested them. The
  kept lock limits changes; a provider that breaks surfaces as a load error or a rollback.
- **Native wheels from `/data`.** A volume mounted `noexec` cannot load them; the provider then shows the import
  error. To check on the HA app and the lab VM in S4.
- **A provider that writes into its own folder** fails the hash at the next start and is reinstalled each time.
  Covered by the developer docs; worth a registry check.
- **Start-up cost.** Community providers load after the index fetch (at most 10 s) and, when the server pins changed,
  two uv resolves (well under a second each when online). Built-in providers and readiness do not wait for it.

