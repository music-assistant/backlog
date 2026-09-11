# Music Assistant OS: technical plan

Status: proposal, 2026-09-10; spike results added 2026-09-11. Owner: Marcel van der Veldt. This document is the technical
companion of the "Music Assistant OS" story on the project board. It is written so it can be fed
to an agent for implementation; every claim marked "verified" was checked in the referenced code.

## Summary

Music Assistant OS (MAOS) is a branded variant of Home Assistant OS (HAOS) that runs the real
Home Assistant Supervisor in a new "appliance mode" (Home Assistant Core never installed or
started), with Music Assistant (MA), the Local Audio App and a small onboarding add-on
preinstalled in the data partition. MA gets a builtin, MAOS-only plugin provider that drives the
Supervisor's REST API (network, OS/app updates, NAS mounts, backups, audio device, add-on store)
and renders Network/Storage/System settings with the existing config-entry UI, so v1 needs no
frontend changes. The OS layer repo `music-assistant/operating-system` pins HAOS as a submodule
and only adds the variant identity, three defconfigs, the data-partition pre-seed and a udev
automount rule. Everything else (A/B updates with rollback, NetworkManager, BlueZ, os-agent,
CONFIG-USB provisioning, backups, add-on store, CLI) is inherited and maintained by HA.

Dependencies to negotiate with the HA team (both projects are OHF): an appliance mode in the
Supervisor (4 tiny patches), a way to point the Supervisor at our version file (hardcoded today),
and a variant/branding flag in HAOS. Each has a fallback (see Risks).

Why not the alternatives (all researched): DietPi/Pi OS/Armbian have no atomic updates or rollback
(a bad update on a retail box is an RMA); balenaOS's good parts (deltas, host OS updates,
dashboard) are cloud-only and openBalena is AGPL and "perpetual beta"; Fedora IoT is podman with
no consumer Wi-Fi story; Ubuntu Core wants an Ubuntu One login at first boot; HiFiBerryOS,
Volumio and moOde are not container based and HiFiBerry abandoned its Buildroot OS in 2025. Our
own supervisor daemon (Go or Python) would rebuild network, updates, mounts, backups and a store
that the HA Supervisor already has. Full HAOS + HA Core with the MA add-on was ruled out.

## Requirements

- Hardware: Raspberry Pi 4/5 (incl. CM5) and generic x86-64 first; box hardware is a decision
  for commercial partners, OHF provides the fundamentals.
- MA only, no HA Core running. Keep a path open to enable HA later.
- Wi-Fi/LAN setup inside the MA UI, plus a first-boot hotspot captive portal, plus Improv Wi-Fi
  over Bluetooth (HA companion app today, the MA app later). USB-stick config alone is not enough.
- Local audio playback in v1 through the existing Local Audio App container.
- Auto-mount of USB storage. Safe OS updates. Branded as Music Assistant OS.
- Box logic outside MA; reuse the HA Supervisor; own Python supervisor as fallback.
- Updates user-triggered with a notification, plus an optional schedule that only runs when
  nothing is playing.
- Nothing may block a later "home OS" base layer (shared users/SSO, ingress, selectable core apps).

## Architecture

### 1. OS layer: a branded HAOS variant, thin repo

- Repo `music-assistant/operating-system`: `operating-system/` = HAOS submodule pinned to a release
  tag; `buildroot-external/` = MAOS external tree (Buildroot supports several, and exports
  `BR2_EXTERNAL_MAOS_PATH` to packages and scripts); `Makefile` + `scripts/enter.sh` copied from
  HAOS with `BR2_EXTERNAL=<haos tree>:<maos tree>`; CI copied from HAOS's `build.yaml`.
- Branding = a HAOS variant, to be upstreamed as a build option (`BR2_HAOS_VARIANT_ID="maos"`,
  `BR2_HAOS_VARIANT_NAME="Music Assistant OS"`, default hostname), consumed by `post-build.sh`
  and `name.sh`. It must keep the CPE `home-assistant:haos` and `SUPERVISOR_MACHINE` untouched
  (the Supervisor recognises the OS by them: `os/manager.py` raises for any other os name and the
  `HAOS` job condition gates mark-good), while changing os-release `NAME`/`PRETTY_NAME`/
  `VARIANT_ID`, `/etc/issue`, motd, the image prefix (`maos_rpi4-64-<ver>.img.xz`) and the RAUC
  compatible (`maos-rpi4-64`, which isolates our update stream). Until upstream lands, our layer
  carries this in copied `post-build.sh`/`post-image.sh` (~65 lines, drift-checked in CI).
- Defconfigs `maos_rpi4_64`, `maos_rpi5_64` (after the rauc-hook fix), `maos_generic_x86_64`:
  copies of HAOS's with the variant set, hostname `musicassistant`, `BR2_PACKAGE_HASSIO` replaced
  by `BR2_PACKAGE_MAOS_DATA`, overlay list = HAOS overlay + MAOS overlay. Board dirs and
  `haos-hook.sh` reused from HAOS unchanged. Partition labels, PARTUUIDs and mount points stay
  HAOS-identical (u-boot, grub, tryboot, expand/data/wipe services and os-agent depend on them).
- `package/maos-data/`: the HAOS `hassio` package pattern (skopeo + docker-in-docker) producing
  `data.ext4` with: supervisor, dns, audio, cli, multicast, observer (versions from HA's channel
  file, exactly like HAOS), the MA image, the Local Audio image, the onboarding add-on image, no
  Core (a tiny dummy image until appliance mode is official); pre-written
  `supervisor/updater.json` (channel + our version URL), `supervisor/homeassistant.json`
  (`boot: false`, `watchdog: false`, appliance flag or the dummy image trick), the installed-add-on
  state (`addons.json` + local add-on configs under `addons/local/`) so the three add-ons are
  installed and start on first boot. Size: truncate 6G then `resize2fs -M` (the MA image is ~700MB
  compressed because of torch, ~2.5GB in the containerd store; expect a ~1.5GB `.img.xz`).
- `rootfs-overlay/`: udev rule `80-maos-media.rules` calling
  `systemd-mount --no-block --collect $devnode /mnt/data/supervisor/media/<label-or-uuid>` (the
  pattern from the systemd-mount man page; PID 1 performs the mount, so udev's private mount
  namespace is not a problem), plus motd/issue. Nothing else on the host changes.
- Kept from HAOS for free: `haos-config` (CONFIG USB/boot-folder import: NM keyfiles, udev,
  modules, `authorized_keys` -> dropbear on 22222, offline `*.raucb` install), expand/swap/zram,
  os-agent (data-disk move, factory wipe, swap, LEDs, Pi firmware), RAUC, NetworkManager, BlueZ,
  udisks2, kernel with USB audio, exFAT/NTFS3/vfat, CIFS/NFS.

### 2. Runtime: the HA Supervisor in appliance mode

- The stock `haos-supervisor` unit starts the Supervisor; the Supervisor starts the three add-ons
  at boot (`startup: initialize` for onboarding, `services` for Local Audio, `application` for MA)
  and marks the RAUC slot good at its own startup (`core.py`, before add-ons and before any Core
  logic; verified single call site). A broken MA does not roll back the OS; the Supervisor's
  add-on watchdog and rollback-to-previous-version cover the app.
- Appliance mode (upstream ask): today `boot: false` alone is not enough because
  `Core.start()`'s `finally` block installs Core whenever the version is `landingpage`, and
  `HomeAssistantCore.load()` starts the landing page unconditionally. Needed patches: gate both on
  `boot` (or a real `appliance` flag), return early in `load()` when nothing is configured, and
  skip the custom-image "unsupported" evaluation in appliance mode. Spike workaround without any
  patch: pre-seed a tiny dummy image and `homeassistant.json` with `version`, `image`,
  `override_image: true`, `boot: false`, `watchdog: false`; `_attach()` accepts an image without a
  container, so Core is never pulled or started; the only effect is an "unsupported: custom image"
  badge, which gates nothing (verified).
- Version source (upstream ask, load-bearing): `URL_HASSIO_VERSION` is a constant in
  `supervisor/const.py`, only the channel is configurable. Ask for a `version_url` key in
  `updater.json` (pre-seeded by us) or a value derived from os-release. Our
  `version.music-assistant.io/{channel}.json` is generated on a schedule from HA's file: same
  supervisor/plugin entries, our `hassos` map (board -> MAOS version) and our `ota` URL template
  (GitHub Releases of the MAOS repo). Supervisor and plugins keep updating from HA's images.
- MA container: the standard image, installed as a separate store entry `music_assistant_os`
  (same image; `hassio_api: true`, `hassio_role: manager`, `host_network`, `audio: true`,
  `map: media:rw`, no ingress/auth_api/homeassistant_api/discovery). Verified: the manager role
  covers `/network/*`, `/os/*` (except datadisk wipe), `/host/*`, `/mounts/*`, `/store/*`,
  `/addons/*`, `/backups/*`, `/audio/*`, `/hardware/*`, `/supervisor/*`; `admin` adds only datadisk
  wipe and add-on security. The standard `music_assistant` add-on keeps its permissions.
- Local Audio: the existing add-on unchanged. Output device = its `audio_output` option through
  `/addons/local_audio/options` + restart (the Supervisor writes the Pulse client config at
  container start). Hot-plugged USB DACs are handled by the Supervisor's audio plugin (udev event
  -> Pulse module reload). ALSA/PipeWire on the host is not needed.
- USB storage: the udev rule mounts under `/mnt/data/supervisor/media/<label>`; add-ons receive
  `/media` with `rslave` propagation (verified in `docker/app.py`), so MA sees `/media/<label>`
  live. NAS shares go through the Supervisor's `/mounts` (CIFS/NFS, host-side systemd automount,
  also under `/media/<name>`), so MA's own in-container SMB/NFS mounting is not needed on MAOS.
- Enable Home Assistant later: `ha core options --boot=true` (or the API) and Core installs.

### 3. Onboarding add-on (`maos_onboarding`, Python, preinstalled)

- `startup: initialize`, `host_network: true`, `host_dbus: true`, bundles dnsmasq (HAOS has none).
- Wired: DHCP, `http://musicassistant.local:8095` shows MA's existing `/setup` page.
- Wi-Fi capable board with no connectivity after ~60s (NetworkManager `Connectivity` and
  `PrimaryConnection` over DBus): scan first and cache, raise an AP `MusicAssistant-XXXX`
  (NM AP mode with manual IPv4, own dnsmasq for DHCP + wildcard DNS; answer `/generate_204`,
  `/hotspot-detect.html`, `/connecttest.txt` with redirects), serve a one-page portal (SSID,
  password, connect). On submit tear the AP down, try for ~45s, re-raise with an error on failure.
  Design reference: balena wifi-connect (Apache-2.0). brcmfmac scans badly while the AP is up,
  hence scan-then-AP.
- Improv Wi-Fi over BLE: BlueZ GATT peripheral over DBus (`GattManager1`, `LEAdvertisingManager1`;
  5 characteristics, 4 RPCs). No Linux device-side implementation exists (verified; the one Go
  hobby project is GPL-3 and dead), so this is new work (~1 week). Once it exists the HA
  companion apps provision the box with zero app work; the MA app can add the Improv SDK.
  Advertise only while unprovisioned and for a bounded window after boot; stop on first success.
- Applies the chosen network through NetworkManager DBus (`dbus-fast`, MIT). Offer it upstream
  later: HAOS lacks any Wi-Fi onboarding without a USB stick.
- DIY flashers keep the HAOS routes: `CONFIG/network/my-network` keyfile on the boot partition or
  a `CONFIG` USB stick. Raspberry Pi Imager customisation does not apply.

### 4. MA-side: builtin `maos` plugin provider (server repo)

Verified against the code; a plugin provider beats a core controller because a manifest that is
simply not registered off-MAOS is invisible everywhere, whereas `CONFIGURABLE_CORE_CONTROLLERS`
(`music_assistant/constants.py`) is a static tuple served to every install.

- Gate: `is_maos()` next to `is_hass_supervisor()` in `music_assistant/helpers/util.py` (env
  `MAOS=1` plus a probe of `GET /os/info` and `GET /core/info` with the Supervisor token),
  `mass.running_as_maos`, a skip in `__load_provider_manifests` (`mass.py`, same shape as the
  `_`-prefix dev-mode skip), a `maos` flag in diagnostics. `ServerInfoMessage.maos` is a later
  models bump.
- `is_hass_supervisor()` keeps returning True on MAOS (its 401 comes from the Supervisor's auth
  middleware, not from Core), so MAOS mode must also: not flip the `hass` provider to builtin
  (`mass.py` ~1500, it would auto-configure against a Core that does not exist), skip the
  Supervisor discovery POST, and skip the ingress site.
- Manifest: `type: plugin`, `builtin: true`, `allow_disable: false`, `multi_instance: false`,
  no extra Python requirements (aiohttp to `http://supervisor` with `SUPERVISOR_TOKEN`),
  `setup_flow.py` for the Wi-Fi picker.
- Modules: `supervisor.py` (REST client behind a small backend interface so a fallback supervisor
  can be swapped in), `storage.py` (reconcile `filesystem_local` instances against `/media`),
  `models.py` (server-local DTOs like `RemoteAccessInfo`), `setup_flow.py` (scan -> SSID select
  with strength -> password -> PROGRESS -> finish; nothing persisted in MA; wrong password
  re-renders the form like `providers/hass/setup_flow.py`).
- API commands registered from `handle_async_init` via `mass.register_api_command` (hass provider
  pattern), all `Scope.SYSTEM_MANAGE`: `maos/info`, `maos/network/{info,wifi/scan,wifi/connect,
  wifi/forget,configure,set_hostname}`, `maos/storage/{list,eject,add_provider,forget}`,
  `maos/mounts/*`, `maos/system/{check_updates,update_os,update_server,reboot,shutdown,backup}`,
  `maos/localaudio/{info,restart,set_output}`. Status pushes via `signal_provider_event`
  (`PROVIDER_EVENT`, object_id `maos/<scope>`); long operations through `controllers/tasks` so
  the Tasks UI shows progress.
- Storage UX: drive appears under `/media/<label>` -> an existing `filesystem_local` with that
  path is loaded immediately, else (opt-out setting `auto_add_usb_drives`) a new instance is
  created through a small public `add_provider_instance()` wrapper around the private
  `_create_provider_instance` in `controllers/config/providers.py`. Drive gone -> nothing
  destructive (the provider self-heals via its 300s availability probe). Eject = unload provider,
  then unmount via the Supervisor host API or an onboarding-app helper. "Forget" =
  `remove_provider_config`.
- Updates: "check" reads `/os/info`, `/addons/self/info`, `/supervisor/info`; "update server" =
  `POST /addons/self/update`; "update OS" = `POST /os/update` then reboot prompt. Optional
  auto-update (off by default): daily window and whether OS updates are included; inside the
  window with no player playing and no active stream/announcement, MA calls the same endpoints,
  otherwise it waits for the next window. MA knows the player state, so the schedule and idle
  check live here; the Supervisor only executes.
- Frontend v1 needs zero changes: provider card + generic entry renderer (LABEL/ALERT status rows
  with `translation_params`, ACTION buttons incl. one `eject_<id>` per drive, two-step confirm for
  reboot/shutdown via the re-render contract, select entries for the Local Audio output device and
  the update channel) + "Reconfigure" for the Wi-Fi flow. Phase 2 adds a Settings > System section
  in the frontend repo (shadcn/Tailwind, no new Vuetify).
- Effort: gating S, provider + client M, storage M, entries + flow M, tests M (fake Supervisor on
  a local aiohttp server), frontend v2 L.
- Network changes drop the client and change the IP MA bound at startup: offer a server restart
  after connecting instead of hot-reloading the webserver.

### 5. Release pipeline

- OS: CI in the MAOS repo (HAOS's `build.yaml`: privileged builder container, ccache, per-board
  matrix) produces `.img.xz` + signed `.raucb` on GitHub Releases; signing key in repo secrets,
  dev builds self-sign (HAOS's `generate-signing-key.sh` flow). OS releases are rare: app,
  Supervisor and plugins update independently after first boot.
- `version.music-assistant.io/{channel}.json`: scheduled job copying HA's `{channel}.json`,
  replacing the `hassos` map and `ota` template. Offline path: `*.raucb` on a CONFIG USB.
- Add-ons: the existing `home-assistant-addon` store repo gets the `music_assistant_os` and
  `maos_onboarding` entries; Local Audio is already there. MA updates ride the normal MA release
  train; the box picks them up like any HAOS install.

### 6. Home-OS compatibility

Choices that keep a later shared base layer (users/SSO, ingress, selectable core apps) open: the
Supervisor stays the app platform (no second supervisor); appliance mode makes the base layer
independent of a particular core app; no box logic inside MA and the `maos` provider's backend is
one small interface; partition and Docker layout stay HAOS-identical; the onboarding add-on is
app-agnostic; MA keeps pluggable login providers and the federation research runs now.

## Phasing

Phase 0, spike on real hardware (~1 week): prove appliance mode and the pre-seed, no OS build.
1. Official HAOS 18.x on a Pi 4 (and an x86 VM): apply `homeassistant.json` with the dummy-image
   trick, reboot, confirm Core never starts, `rauc status` shows the booted slot good and
   `BOOT_A_LEFT` back to 3, `ha os info`/`ha network info` work.
2. Install MA from a local add-on dir with the `music_assistant_os` manifest, Local Audio from the
   store; from inside MA call `/network/info`, `/os/info`, `/mounts`, `/addons/local_audio/options`.
3. udev automount rule via the CONFIG USB `udev/` folder: a stick appears at `/media/<label>` in
   MA; a Supervisor CIFS mount appears the same way. Local Audio plays; a USB DAC hot-plug works.
4. Hotspot prototype in a `host_dbus` container: NM AP mode + manual IPv4 + dnsmasq; captive
   portal probes answered on an iPhone and an Android phone.
5. Build the pre-seeded `data.ext4` in CI with the `hassio` package pattern and repack a HAOS
   image with it; first boot offline reaches the MA UI.
6. In parallel, open the upstream discussions with the spike results.

Phase 1, product (~2-3 months): appliance mode + version URL in the Supervisor and the variant flag
in HAOS (or our carried copies), the MAOS variant build + signing + version file, onboarding add-on
(portal + Improv), `maos` provider + Wi-Fi flow in MA, `music_assistant_os` store entry, rpi5/CM5,
docs for DIY flashing. Exit: a non-technical user unboxes, provisions Wi-Fi from a phone, plays
music from a USB stick and a streaming service, and installs an OS and an app update from the MA UI.

Phase 2, retail polish: Settings > System pages in the frontend, `config.txt`/dtoverlay
persistence for I2S DAC HATs (boot partition edits survive the generic rauc-hook path on Pi 4; Pi 5
slot layout leaves root `config.txt` alone; needs a UI/config fragment), Bluetooth audio through
the Supervisor audio plugin (investigate), data-disk move and factory reset (`admin` role for wipe),
board LEDs, backups to NAS from the MA UI, "enable Home Assistant" switch.

## Upstream asks (to raise with the HA/HAOS team)

Supervisor (appliance mode, ~4 small patches + 1 decision):
1. `supervisor/core.py` ~344: gate the background landing-page install on `boot` (or an
   `appliance` flag).
2. `supervisor/homeassistant/core.py` ~133: gate the landing-page start on `boot`; ~97: return
   early in `load()` when `boot` is off and nothing is configured.
3. `resolution/evaluations/home_assistant_core_custom_image.py`: skip in appliance mode.
4. Load-bearing decision: a configurable version file (`version_url` in `updater.json`, or derived
   from os-release), so a variant can ship its own `hassos` map and OTA URL. Alternative: HA's own
   pipeline builds and hosts the `maos-*` targets as pseudo-boards in their `hassos` map.
5. Optional: check RAUC `compatible` before install; make `/backups/freeze` not require Core;
   silence the Core-absent warnings in ingress.

HAOS:
6. Variant/branding build option (`BR2_HAOS_VARIANT_*`) consumed by `post-build.sh`/`name.sh`,
   keeping the CPE and `SUPERVISOR_MACHINE` intact.
7. `ota/rauc-hook`: match `*-rpi5-64` instead of the hardcoded `haos-rpi5-64` compatible.
8. `create-data-partition.sh`: take the image list and extra data files as arguments.
9. Offers back: udev automount of USB drives into the Supervisor media dir; dnsmasq package; later
   the onboarding add-on as an official Wi-Fi onboarding path.

Also to settle with OHF: branding ("Music Assistant OS, based on Home Assistant OS"; the tty
console shows the HA CLI banner unless the cli plugin becomes variant-aware) and the GPL
source-offer obligation for a distributed image (public repo + Buildroot sources, as HAOS does).

## Risks and fallbacks

- HA team declines appliance mode: our own Python supervisor container on the same layered build
  (reusing the Supervisor's Apache-2.0 DBus modules for NetworkManager, RAUC, udisks2, logind,
  hostname); the `maos` provider's backend interface hides the difference.
- HA team declines the version-URL change and will not host our targets: ship unbranded OS
  partitions (`haos` compatible, official HAOS OTA) with branding limited to hostname and the MA
  UI, or carry a patched Supervisor image (not recommended: it self-updates from the version file).
- The Supervisor self-updates before appliance mode is official and a release breaks the dummy
  image trick. Pinning it is not an option: its job conditions block app installs and updates
  whenever the Supervisor is outdated. Accept the risk, watch Supervisor releases, and get
  appliance mode upstream quickly.
- HA architecture discussion #1447 (Aug 2026) proposes tightening add-on roles: our `manager`
  dependence sits in its path; being a first-party OHF use case is the mitigation.
- RAM: Supervisor + plugins add ~300MB on top of MA (2GB floor, 4GB recommended): Pi 4 4GB / Pi 5.
- brcmfmac AP/STA and BLE coexistence during onboarding: scan-then-AP, bounded retries.
- Improv is open by design (no button): bounded advertising window after boot.
- Image size ~1.5GB xz because of torch in the MA image (an optional analysis extra would halve it).
- Partner hardware not chosen: an RK3566-class board would be added like HAOS-CN did.

## Side findings (follow-ups outside this plan)

- `home-assistant-addon`: `local_audio/config.yaml` ships `apparmor.txt` but never sets
  `apparmor: true`; `music_assistant_nightly` also lacks the `apparmor:` key; `repository.json`
  points at the non-existent `music-assistant/core`; README links issues to the deprecated
  `hass-music-assistant`; stray `gitignore` file at the root.
- `music_assistant/helpers/pulse_capture.py` docstring references a removed `enumerate_pa_sinks()`.

## Spike results (2026-09-11, Home Assistant OS 18.2 generic-aarch64 in UTM)

Everything below is verified on a VM; the Pi and the Wi-Fi hotspot are still open (needs hardware).

- Core-less Supervisor: with a dummy Core image carrying the labels `io.hass.version=2099.1.0`,
  `io.hass.arch=<arch>`, `io.hass.type=core` and `homeassistant.json` set to `boot: false`,
  `watchdog: false`, `override_image: true`, `version: 2099.1.0`, the Supervisor attaches the
  image, logs "Skipping start of Home Assistant", marks the RAUC slot good at its own startup
  (grub `A_OK=1`, `A_TRY=0` after reboot), stays `healthy: true`, and the only unsupported reason
  is `home_assistant_core_custom_image`. Without the version label the Supervisor reads version
  `None` and its Core loader throws in a background task.
- The Supervisor renamed add-ons to apps: local apps live in `apps/local/<dir>` (slug
  `local_<dir>`, containers `app_<slug>`), installed state is `apps.json` with a `user` and a
  `system` section, the CLI is `ha apps ...`, and the store needs `ha store reload` to pick up a
  new local app. The Music Assistant add-on repository is a default Supervisor repository.
- Manager role from inside Music Assistant: `/network/*`, `/os/*`, `/host/*`, `/mounts/*`,
  `/audio/*`, `/hardware/*`, `/store/*`, `/addons/*`, `/backups/*`, `/supervisor/*`,
  `/core/info`, `/os/datadisk/list` answer 200; `/os/ssh/authorized_keys` is 403. Music Assistant
  configured and started the Samba app and created a CIFS media mount through the API; the mount
  appeared live under `/media/nas` inside its own running container.
- USB automount: two udev rules on the persistent overlay (`RUN+=systemd-mount --no-block
  --collect $devnode /mnt/data/supervisor/media/<label-or-uuid>`) mounted an exFAT stick within a
  second; Music Assistant listed its files live. On unplug systemd unmounts, but the empty mount
  point stays, so matching `ACTION=="remove"` rules `rmdir` it, and the MAOS provider must test
  for a mounted filesystem, not a directory.
- Local Audio starts and registers as a Sendspin player even without a sound card; playback and
  USB DAC hot-plug remain to be tested on hardware.
- Music Assistant under the core-less Supervisor logs two things the MAOS mode must suppress: the
  discovery announcement (403, outside the manager role) and the Home Assistant provider retrying
  the Core websocket every two minutes (502).
- Pre-seeded image (music-assistant/operating-system, PR #1 merged): the data partition is built
  with the same dockerd as the device (29.6.2, containerd snapshotter) inside a privileged builder
  container; `docker load` unpacks only native-arch layers, so the aarch64 partition is 4.7GiB when
  built on arm64 and about 1.9GiB when built on amd64 runners (layers unpack at the first container
  start on the device; stock HAOS ships the same way). The repack replaces the last GPT partition
  keeping its GUID and the Raspberry Pi hybrid MBR; `maos_generic-aarch64-18.2.img.xz` is 1.7GB.
  Offline first boot: 27s to Docker, slot marked good at 29s, Local Audio at 29s, Music Assistant
  at 36s, UI within a minute, nothing downloaded. The hostname is pre-seeded on the overlay
  partition (`musicassistant.local`); os-release and the console still say Home Assistant until the
  variant build exists. CI builds the data partitions and the rpi4, rpi5 and generic-aarch64 images
  on GitHub runners.
- Sizes on the device after real use: server image 3GB unpacked, Supervisor 0.5GB, plugins 0.4GB,
  Local Audio 0.4GB; the containerd store keeps compressed and unpacked layers, 7.3GB in total.
- Every device flashed from one image shares the pre-seeded Supervisor and app UUIDs (the app
  access tokens are regenerated on every app start); first boot should regenerate them later.

## Facts established (2026-09-09/10, condensed, verified in code or docs)

MA packaging: `ghcr.io/music-assistant/server`, Debian trixie + Python 3.14, amd64/arm64, rolling
`stable`/`beta`/`nightly` tags plus immutable version tags; needs host networking, root +
SYS_ADMIN + DAC_READ_SEARCH only for its own in-container SMB/NFS mounting; `/data` volume,
`/media` default local path; no host integration code; `/setup` onboarding exists;
`is_hass_supervisor()` = `SUPERVISOR_TOKEN` + a 401 from `http://supervisor/core`. Add-on config:
`host_network`, `privileged` caps, `audio: true`, `map: media:rw, ssl:ro`, ingress 8094. Local
playback provider retired (PR #5965) for `music-assistant/local-audio-addon` (sendspin-cpp-cli;
ALSA/Pulse/PipeWire; `audio: true` under the Supervisor; standalone docker with `/dev/snd`).

HAOS dev: Buildroot 2025.02 LTS fork, `HAOS` external tree, `meta` drives id/name/RAUC
compatible, 14 64-bit boards, `hassio` package pre-seeds supervisor/dns/audio/cli/multicast/
observer + landing page and writes `updater.json` (not `homeassistant.json`), unconditional
overlay with `haos-supervisor`, no flavor mechanism, preset files can disable units, post-build
sources HAOS `meta` then board `meta`, rauc-hook hardcodes `haos-rpi5-64`, root erofs read-only,
NM shared mode needs the dnsmasq binary (absent), no PulseAudio/PipeWire/alsa-utils on the host,
`haos-config` CONFIG import is Supervisor-independent, os-agent Apache-2.0. Existing derivatives
(HAOS-CN, haos-ace, mixtile) only add boards. Licensing: Apache-2.0 projects, GPL components in
the image; "Home Assistant" is an OHF trademark.

Supervisor main (2026-09-07): `mark_healthy` called once in `Core.start()` before add-ons and Core;
`UnhealthyReason` has nothing Core-related; `HOME_ASSISTANT_CORE_SUPPORTED` no-ops for absent/
landing-page Core; `os update`, add-on install/update, network, mounts do not depend on Core;
`boot` exists (`ha core options --boot=false`) but the `finally` install and the landing-page
start bypass it; `_attach()` accepts a bare image; `hassio_api` + `manager` covers the control
plane; `/media` and `/share` bound `rslave` from `/mnt/data/supervisor/{media,share}`;
`URL_HASSIO_VERSION` hardcoded, channel-only; OS update does not check RAUC `compatible`
(RAUC does); `audio_output` per add-on written at container start; USB DAC hot-plug reloads Pulse;
`/backups/freeze` hard-requires Core. Prior art: architecture#412 (2020, closed without
engagement), MA #5154 (unanswered), architecture#1447 (role tightening, Aug 2026).

Alternatives: balenaOS (openBalena AGPL, no deltas/host updates), DietPi (apt, single maintainer),
Pi OS Lite (Pi only, cloud-init), HiFiBerryOS (unmaintained 2025), Fedora IoT/CoreOS, Ubuntu Core,
Armbian, Volumio/moOde/piCorePlayer/RoPieee: rejected. Onboarding blocks: wifi-connect (Rust,
Apache-2.0), Comitup (Python, GPL-2), NM hotspot + own portal; Improv BLE has client SDKs (HA apps
ship them) and no Linux device side. USB automount: udev + `systemd-mount`; docker `rslave` bind
propagation for live hot-plug.

## Verification (Phase 0 exit criteria)

1. Pi 4 + x86 VM on official HAOS with the pre-seeded `homeassistant.json`: Core never installed
   or started after two reboots; `rauc status` booted slot good; `BOOT_A_LEFT` = 3.
2. MA (OS-edition manifest) answers on `:8095`; from inside MA: `/network/info`, `/os/info`,
   `/mounts`, `/addons/local_audio/options` return 200; `/os/ssh/authorized_keys` returns 403.
3. USB stick and a Supervisor CIFS mount both appear under `/media` inside MA without a restart;
   Local Audio plays; a hot-plugged USB DAC becomes selectable.
4. Hotspot prototype: phone joins `MusicAssistant-XXXX`, portal pops up, home Wi-Fi joins.
5. Repacked image with the pre-seeded data partition reaches the MA UI on first boot offline.

