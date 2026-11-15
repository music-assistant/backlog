# Squeezebox and squeezelite speakers in Sendspin groups: technical plan

Status: planned 2026-11-15. Work starts once the player split (server#7649) is on `dev`. Owner: Marcel van der Veldt. This document is the technical companion of the "Squeezebox and squeezelite speakers play in sync with Sendspin speakers" epic (#463) on the project board. It is written so it can be fed to an agent for implementation.

Claims marked "verified" were checked in the referenced code:
- server `dev` at `6e33e7f3b` and `feature/player-split`;
- aioslimproto `main` at `bfd673e`;
- aiosendspin `main` at `6cf131b`;
- squeezelite, squeezelite-esp32, SqueezePlay and Lyrion Music Server (LMS) sources.

Claims marked "inferred" come from reading how LMS drives closed firmware.

## Summary

Speakers on the Squeezebox protocol (slimproto: squeezelite, Squeezebox hardware, WiiM/LinkPlay) only sync with each other. Since server#6477 that sync follows the way LMS does it (aioslimproto `SyncGroup`), and its corrections are skips or pauses of 10 ms or more. These speakers cannot join a group with Sendspin speakers.

The fix is a Sendspin bridge in the squeezelite provider, built like the AirPlay one. A slimproto player registers as an external Sendspin client. In a Sendspin group the bridge:
- plays the group's timeline on the speaker through one endless PCM stream per session;
- starts the speaker at the exact instant (`strm u` with a jiffies time);
- keeps the stream about 1.5 s ahead of what the speaker plays;
- holds the speaker on the timeline by adding or dropping single frames in the audio it sends, based on the position the speaker reports about four times a second.

Once that holds on real speakers, all grouped playback of these speakers moves to Sendspin. The old multi-client group path then goes.

## Decisions

Taken with the maintainer on 2026-11-14 and 2026-11-15:

- No code, aioslimproto included, until the player split is on `dev`. The bridge is written against the split's protocol API.
- A device carries one Sendspin output. A WiiM also speaks AirPlay, and its AirPlay bridge wins (as AirPlay already does over Cast) until both bridges have been measured on it in #468. The better one wins after that.
- Sendspin stays on `CLOCK_MONOTONIC_RAW`. aioslimproto gets a clock the caller passes in, and the bridge passes Sendspin's (see "Clock domain").
- Target: all grouped playback of these speakers through Sendspin, removing the multi-client group stream (#469). Before that, the bridge has to match today's group quality (#467). Playing on a single speaker stays on the native path.
- No microphone test. Latency a speaker does not report (HDMI or USB DACs, amplifiers, Bluetooth) is covered by the existing per-player sync delay setting.

## Requirements

- A slimproto speaker in a Sendspin group stays within about 1 ms of the group after settling, with corrections nobody can hear.
- A start, seek, resume and late join land on the group's timeline; a speaker's constant start offset is learned once.
- Volume, mute, buttons, remotes and app commands keep working. A command from the speaker goes to the group.
- Underruns, reconnects and sessions longer than a day recover by themselves.
- A squeezelite-only group keeps working throughout. Until #469 it stays on the native path.
- No new way to obtain provider audio. The bridge stream is PCM for one speaker on the local network, served by the stream server like every other player stream.

## Measurements so far

Spikes of 2026-11-13. A test slimproto server ran an endless PCM stream with frame-level correction. Ground truth came from markers in the audio (lab) or the speaker's own reports (hardware).

| Setup | Hold after settling | Start |
|---|---|---|
| Lab, virtual squeezelite, +100 / −60 ppm forced drift | ±0.3 ms p95, max 0.55 ms | sd ~0.8 ms over 10 starts |
| squeezelite on macOS (CoreAudio), over the LAN | p95 0.17–0.30 ms, max ~0.5 ms | 21–31 ms late (output buffer); ~2.5 ms after compensation |
| WiiM Pro, over the LAN | 0.09 ms median, 0.36 ms max, 36 s settle | 420 ms late but constant; 8.8 ms after compensation |

- **Corrections:** one frame at a time, at least 83 ms apart, 0.1–4.5 per second. Real DAC drift was −4 to −7 ppm; wander around a straight line was p95 0.2–0.3 ms.
- **Report noise:** sd 0.3–0.4 ms on hardware, 1.2 ms in the lab VM. Reports are whole milliseconds.
- **Native path, for comparison** (lab, after the fixes in aioslimproto#558): about 6 ms p50 in steady state. On real speakers a start or resume landed 30–200 ms apart until the first correction about 5 s later.

## Devices

From the source research (verified unless marked):

| Family | Full sync | Notes |
|---|---|---|
| squeezelite on Linux/ALSA, piCorePlayer | yes | Jiffies on `CLOCK_MONOTONIC`. `elapsed` subtracts `snd_pcm_delay`, so the start error (about the ALSA ring, ~40 ms) can be measured. Targets more than 10 s ahead start at once (`output.c:99-110`). |
| squeezelite on macOS | yes | Jiffies follow the wall clock (`gettimeofday`, `utils.c:117-119`); a slew of 909 ppm for a minute was seen. Corrections pause while the clock jumps. |
| squeezelite on Windows | yes, noisier | `GetTickCount`, ~16 ms resolution. |
| squeezelite-esp32 with an I2S or SPDIF DAC | yes | Monotonic `esp_timer` jiffies. The DMA delay is estimated and compensated at start; up to ~12 ms of it is not visible in `elapsed`. Start at least 300 ms ahead. Headless model reports device id 101. |
| squeezelite-esp32 with Bluetooth output | drift only | `device_frames` is 0, so the Bluetooth latency is invisible; the sync delay setting covers it. |
| Squeezebox Radio, Touch (SqueezePlay) | yes | `elapsed` is DAC based. The start must be at most 5 s ahead (`decode_alsa_backend.c:304`). The start also waits for `output_threshold` of decoded audio; the default 20 (2 s) deadlocks a 1.5 s lead. The player pauses itself after an underrun. |
| Squeezebox Controller | yes | Same SqueezePlay audio code. It is only a player with its "Audio playback" beta setting on, which is off by default. |
| SB2/SB3/Classic, Receiver, Boom, Transporter | yes (inferred) | Closed firmware. LMS starts them with a timed `strm u` and trusts their play points (10 ms deadband). |
| WiiM / LinkPlay | yes (measured) | HELO says squeezelite, device id 12, `AccuratePlayPoints=1`. |
| First Squeezebox (SB1) | no | No timed start: LMS fakes `startAt` with a server timer and a plain unpause (`Squeezebox1.pm:335`), and estimates its position from byte counts (`SB1SliMP3Sync.pm`). |
| SliMP3 | not an MA player | Uses its own UDP protocol, which aioslimproto does not speak. |

Device id 12 covers squeezelite, SqueezePlay and WiiM, so behaviour per family keys on the HELO `Model=` capability.

## How it works today

**Sendspin bridges (verified).** `SendspinBridgeManagerBase` (`providers/sendspin/bridge_manager.py`) reconciles bridge lifecycle. A provider supplies `_bridge_client_id`, `_create_bridge` and `_should_have_bridge`. `BridgePlayerRole` (`providers/sendspin/bridge_role.py`) receives `AudioChunk(data, timestamp_us, duration_us)` and provides:
- volume, mute, stream start/end and explicit-stop callbacks;
- `setup_audio_requirements` for the PCM format;
- `set_timing(required_lead_time_ms, min_buffer_ms)`.

On the player split, everything is keyed by protocol id, and `register_bridge_underlying_protocol` replaces `register_bridge_underlying_player`.

**The AirPlay bridge (verified, `providers/airplay/sendspin_bridge.py`).** The model for the audio path:
- anchors byte 0 on the first chunk's timestamp;
- places every chunk at the byte offset its timestamp claims (`_align_chunk`), filling holes with silence and trimming overlaps;
- paces the writer to at most 8 s ahead (`_cli_writer`).

It is open loop after the anchor, because the AirPlay receiver follows the shared PTP clock. A Squeezebox follows its own DAC crystal, so the slimproto bridge needs a feedback loop.

**Protocol selection (verified, `controllers/players/protocol_linking.py`, `constants.py`).**
- Priorities: airplay 10, squeezelite 20, chromecast 30, sendspin 40, dlna 50.
- A squeezelite-only group therefore uses native grouping. A group with Sendspin players uses the Sendspin output.
- Cast hard-defers its bridge to AirPlay on the same device (`chromecast/sendspin_bridge.py`, `_should_have_bridge`, `_has_foreign_bridge_client`).
- On the split, a player carries one protocol per domain (`_resolve_derived_player_id`).

**squeezelite group playback (verified).** `MultiClientStream` (`providers/squeezelite/multi_client_stream.py`) is served at `/slimproto/multi`, with each player's own output format from `get_output_format` (FLAC, up to the player's maximum rate). The start goes through `_handle_buffer_ready` and `SyncGroup.start`; drift checks run on heartbeats (`SyncGroup.check`); the resume uses `SyncGroup.resume`. `CONF_SYNC_ADJUST` becomes `client.play_delay`.

**Commands from the speaker (verified).**
- Buttons and IR arrive as aioslimproto events (`_handle_player_cli_event`).
- WiiM-style devices send LMS CLI JSON-RPC to `/jsonrpc.js` (`_handle_cli_command`). A WiiM's `stop` is the same request for a user stop, a source switch and another app taking over, so it is never a group stop.

**aioslimproto (verified, `client.py`).**
- `play_url` already takes an explicit PCM format (`audio/pcm;rate=…;bitrate=…;channels=…`, no WAV header), `stream_threshold` and `output_threshold`.
- `strm t` is only sent from the 5 s heartbeat.
- `_store_status` drops `bytes_received` and `output_buffer_fullness`.
- Times are stamped with `time.monotonic()` at socket read.
- `track_jiffies_epoch` (one-way minimum filter), `jiffies_to_timestamp`, `play_point` and `start_at` exist.
- Every HELO sends the stored power state, which is off for a new client. A reconnect therefore turns squeezelite's output off.

**squeezelite reports (verified, `slimproto.c`).**
- STAT fields are whole milliseconds. `elapsed` is `(frames_played − device_frames)/rate`, extrapolated on wall time from a status copy refreshed about every 100 ms.
- During the first device delay after a start it counts from the first frame written instead.
- `strm t` is answered at any rate and echoes our timestamp in the reply.
- The frame counter is 32-bit: it wraps after 27 h at 44.1 kHz.
- Players read greedily (stream buffers: 2 MB desktop, 3 MB SqueezePlay, 500 KB ESP32), so only the server can pace.

## Clock domain

- aiosendspin's server clock is `CLOCK_MONOTONIC_RAW` on Linux (`aiosendspin/clock.py`, aiosendspin#211). aioslimproto uses `time.monotonic()`, which is `CLOCK_MONOTONIC`, rate-corrected by NTP. The two drift apart by the host's clock error, often 10–50 ppm. A bridge that mixes them follows the wrong clock by that much.
- Moving Sendspin to `CLOCK_MONOTONIC` was considered and rejected:
  - every host slew would hit every Sendspin player at once (on HAOS, timesyncd moved `CLOCK_MONOTONIC` 31 ms against RAW within 75 s, sendspin-cpp#208);
  - the reference time filter loses about 0.1–0.7 ms per ppm of rate change.
- The AirPlay drift in airplay-cli#99 is handled in airplay-cli: standalone HomePods play on their own clock there, and RAW is not the cause.
- Decision: aioslimproto takes a clock from the caller (microseconds, like `aiosendspin.clock.Clock`), and the bridge passes `sendspin_server.clock.now_us`. That also keeps host NTP slews out of aioslimproto's own group sync.

## Design

### Where it lives, and when it streams

- `providers/squeezelite/sendspin_bridge.py`: a `SendspinBridgeManager` subclass and a bridge per player, on the split's protocol API. The client id is derived from the MAC, as for AirPlay.
- `_should_have_bridge` requires:
  - a MAC address;
  - a family that can start at a set time (not the SB1);
  - no foreign Sendspin client for the device (the Cast pattern, which keeps the WiiM on its AirPlay bridge).
- Until #469, group selection is unchanged. The bridge streams only when a squeezelite player is grouped with Sendspin players, or when the user picks the Sendspin output.

### Session transport

For each Sendspin stream (start, seek, resume):

1. Anchor the session on the first chunk's `timestamp_us`, as AirPlay does.
2. Power on if needed, send `strm q`, then `play_url` with:
   - `/slimproto/bridge/<player>/<session>.pcm` on a dynamic route (`mass.streams.register_dynamic_route`);
   - `autostart=False`;
   - PCM 44.1 kHz/16-bit stereo with an explicit format;
   - a small `stream_threshold` (32–64 KB);
   - an `output_threshold` below the lead (0–5 tenths).
3. The route serves a plain endless body: no chunked transfer encoding and no Content-Length, because no client decodes chunked bodies.
4. On buffer ready (STMl), `start_at(audible instant − learned start offset − sync delay)`, at least 300 ms ahead (ESP32) and at most 5 s ahead (SqueezePlay).
5. The writer keeps the speaker about 1.5 s ahead of its audible position. It measures the real lead from `bytes_received` and `output_buffer_fullness`, not from bytes written.
   - Without pacing a speaker buffers up to 20 s, and every correction is heard only that much later.
   - This lead is also the delay before a correction is heard, so it stays far below the AirPlay bridge's 8 s.
6. `set_timing`: `required_lead_time_ms` covers a cold start, about 1500 ms (to be measured), and `min_buffer_ms` is the pacing lead.
7. When the stream ends, a short grace period follows, then `strm q`.
   - A seek or resume is a new cold start of about 1 s.
   - Keeping the stream running across a track change or seek is a later optimisation.

### Sync loop (`providers/squeezelite/bridge_sync.py`, without I/O)

- **Measurement.** Each status report gives the speaker's jiffies, mapped through the epoch to a Sendspin time, and `elapsed`, converted to the frame index playing.
- **Ledger.** Every inserted or dropped frame is recorded by stream frame index, which gives the timeline instant of the frame playing now.
- **Error.** Error = Sendspin time − intended audible instant + sync delay.
- **Estimator.** A regression over 60–120 s for drift (ppm) and offset.
- **Control.** A deadband of about 0.5 ms; single frames at least 80 ms apart; at most 500 ppm.
- **Settling.** Reports are ignored for about 1 s after a start, longer for a speaker with a large output delay, while `elapsed` still counts from the first write.
- **Start offset.** After settling, a remaining offset above a threshold is corrected once in a single jump (one short skip). It is stored per speaker and subtracted from `start_at` on later starts. A 420 ms offset is never chased at 500 ppm, which would take 14 minutes.
- **Clock jumps.** When the epoch jumps (squeezelite on macOS), corrections pause until the estimate is stable again.

### Commands, volume, state

- Volume and mute go out as `audg` through the `BridgePlayerRole` callbacks. The speaker's own changes are adopted with `update_player_state`. `aude(0)` is never sent during a session, because it closes squeezelite's output.
- Buttons, IR and LMS CLI pause/play go to the Sendspin group while the bridge owns the stream (like AirPlay's `get_transport_command_target`). A WiiM `stop` is never a group stop.
- An explicit stop or removal from the group (`notify_explicit_stop`) sends `strm q` at once.

### Failure paths

- **Underrun (STMo).** squeezelite plays on later and shifts the timeline by the gap; SqueezePlay pauses itself. The bridge restarts the session at the current timeline.
- **Reconnect (new HELO).** The new client has no epoch and is powered off. The bridge treats it as a fresh start.
- **Long sessions.** A session is restarted, or the counter unwrapped, before the 32-bit frame counter wraps.
- **A speaker that keeps dropping.** It leaves the Sendspin session, with the guard the AirPlay bridge uses (`BRIDGE_TRANSPORT_RECOVERY_GUARD_SECONDS`).

## Work items

The sub-issues of #463, in order:

1. **#464 aioslimproto:**
   - status requests on demand;
   - the raw status fields with the receive time;
   - the caller's clock for stamps, epoch, `start_at` and `SyncGroup`;
   - device id 101;
   - a release and the server bump.
2. **#465 Server:** the sync loop as a module without I/O, with tests that replay the earlier measurement logs.
3. **#466 Server:** the bridge (manager, session transport, route, commands, failure paths), with a lab gate and a hardware test.
4. **#467 Server:** what #469 needs from the bridge:
   - FLAC and the speaker's maximum sample rate, encoded after the correction (PyAV is already a Sendspin dependency);
   - now-playing on Squeezebox screens per track;
   - SqueezePlay specifics;
   - learned start offsets kept across restarts.
5. **#468:** a test on real speakers (squeezelite on macOS and Linux, WiiM Pro, squeezelite-esp32, next to a Voice PE), and the AirPlay versus slimproto bridge on the WiiM. Beta testers for the Radio, Touch and older Squeezeboxes.
6. **#469 Server:** all grouped playback through Sendspin.
   - The squeezelite protocol stops offering native grouping (`SET_MEMBERS`), so groups fall through to the Sendspin output.
   - The multi-client stream, `/slimproto/multi` and the `SyncGroup` calls are removed.
   - The SB1 can no longer join groups. `SyncGroup` stays in aioslimproto for other users.

## Validation

- **Lab:** the existing slimproto rig (virtual squeezelite players, a marker pacer as ground truth) plus a virtual Sendspin player as the reference timeline. Scenarios: start, seek, pause/resume, late join, ±100 ppm, underrun, reconnect.
- **Hardware:** as in #468.
- **The end-to-end lab gate** for the squeezelite and Sendspin areas on every server PR.

## Risks and open points

- **What you actually hear.** `elapsed` only knows the delay a driver reports. The sync delay setting covers the rest.
- **Older Squeezebox hardware** runs closed firmware and is not tested by the team. Ask beta testers before #469 lands.
- **SqueezePlay:** status latency under UI load, and recovery from its self-pause.
- **squeezelite-esp32:** Wi-Fi headroom with a 1.5 s lead and its 500 KB stream buffer.
- **The WiiM:** which bridge wins is settled in #468.
