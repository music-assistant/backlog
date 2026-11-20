# Multi-channel audio: surround sources on players that can play them

Status: decided with the maintainer on 2026-11-20. Story: #494. Owner: Marcel van der Veldt. This
is the technical companion of the story on the project board, written so it can be fed to an
agent for implementation.

Claims marked "verified" were checked on server `dev` at `85136fcbd`, models `1.1.248`,
aiosendspin `10.3.0` (`fcc0125`), the Sendspin spec `main` at `2cfe197`, sendspin-cpp `main` at
`a10289a` and sendspin-cpp-cli at `715f988`. "Measured" means run with FFmpeg 9.0.2 on synthetic
audio with a different tone per channel. Paths are relative to `music_assistant/` in the server
unless they name another repository. A second reviewer checked the design of steps 1 to 4
against the code; what it found is worked in.

## Problem

Music Assistant plays every source as stereo.

- A source with more than two channels is mixed down at the first decode:
  `helpers/audio.py:831,837` (`channels=min(arriving.channels, 2)`), which sets the format of the
  audio buffer. The same cap sits on the format of one item
  (`controllers/streams/audio.py:1631`, and `:1634` for crossfade and overlays), of the flow
  stream (`:1727`), of live sources (`:6263`) and of the output to a player (`:1579`: one or two
  channels). Verified.
- No player can say that it plays more than two channels. A player declares sample rates and bit
  depths (`models/player_protocol.py:1300`) and nothing else. Verified.
- So a 5.1 album plays as stereo on a surround sound card or an AV receiver. A community provider
  (music-assistant/community-catalog#46) works around it by reading a multi-channel local file
  from disk itself. That only works for local files and skips DSP, volume normalisation,
  crossfade and gapless playback.
- Users ask for it: music-assistant/support#5197 (16 votes) and music-assistant/support#4732
  (8 votes). An earlier attempt inside the old local audio provider, music-assistant/server#4919,
  was closed when that provider was retired (music-assistant/server#5965).

## Concept

A player says which **layout** it wants its audio in, and gets every stream in that layout.

- The layout is stereo unless the player declares 5.1 or 7.1. A player that declares nothing gets
  exactly what it gets today.
- A stream to a surround player always has the same channel count and channel order: the order of
  FLAC and WAVE (front left, front right, centre, LFE, back left, back right, then side left and
  side right for 7.1). It does not change between tracks, so gapless playback, crossfade and flow
  mode work as they do for stereo.
- A source with fewer channels is placed in that layout. A source with more is folded into it,
  at the level a surround source has on a stereo player today.
- Inside the server audio travels in one of three widths: stereo, 5.1 or 7.1. `AudioFormat` gets
  no new field: six channels are always 5.1 and eight are always 7.1, which is also what FFmpeg
  assumes for raw audio (measured).
- Everything on the normal path keeps working for a surround stream: buffer, seek, pacing, volume
  normalisation, crossfade, and every DSP filter that does not depend on left and right.

## Decisions (2026-11-20)

1. General support for every player that can take surround, starting with a sound card or HDMI
   output through a provider. That lets the community provider stop reading files itself.
2. Sources: everything that already reports its channel count. Own files (`helpers/tags.py:737`),
   Plex (`providers/plex/__init__.py:1122`), Jellyfin (`providers/jellyfin/parsers.py:206`), Emby
   (`providers/emby/parsers.py:442`), OpenSubsonic (`providers/opensubsonic/parsers.py:161`),
   URLs and live sources. Streaming services stay stereo: Tidal asks for the stereo version on
   purpose (`providers/tidal/streaming.py:193`). Whether the Atmos version of a Tidal track may
   be used at all is research for later, not a build step.
3. Groups: each member gets what it can play, the surround player surround and the others the
   downmix. In step 1 a group stays stereo; surround in a group comes with the Sendspin step.
4. DSP: what does not depend on left and right keeps working on a surround stream. Settings for
   front left and front right apply to the front pair. Balance, stereo width and crossfeed are
   skipped. DSP per centre, surround and subwoofer is a later step.
5. A built-in upmixer for stereo sources: a select on a surround player, with "front speakers
   only" as the default.
6. Order: the core and a provider that declares surround (step 1), the upmixer (2), players that
   fetch the stream themselves (3), Sendspin sessions and combined surround sets (4), a Sendspin
   player that plays surround itself (5), DSP per channel (6).

## What FFmpeg does (measured)

| Case | Result |
|---|---|
| Raw PCM with 6 or 8 channels and no named layout | read as 5.1 and 7.1 |
| A 5.1 source with side surrounds, or a 6.0 source, decoded with `-ac 6` | lands in 5.1: the side pair on the back pair at full level, a back centre split over the back pair |
| Stereo into 6 channels (`-ac 6`) | front left and right carry it, the rest is silent |
| Mono into 5.1 | the centre only; duplicated to stereo first it stays on the front pair |
| Quad into 5.1 | front and back pairs filled, centre and LFE silent |
| 5.1 into 7.1 | six channels in place, the side pair silent |
| 5.1 into stereo, into an integer format (today's downmix) | centre and surrounds mixed in 3 dB down, the LFE dropped, the whole scaled down to stay clip-safe: 4.65 dB below one source channel |
| 5.1 into stereo, into a float format | the same mix NOT scaled down: 7.65 dB louder, and it can clip |
| The same with `aresample=ochl=stereo:rematrix_maxval=1` | the scaled-down level in every format |
| 6-channel FLAC and WAV output | FLAC marks 5.1; WAV gets the extensible header, also on a pipe |
| `equalizer`, `bass`, `treble`, `acompressor`, `alimiter`, `loudnorm`, `atempo`, `acrossfade`, `amix` on 5.1 | stay 5.1 |
| `pan` with a gain per channel on 5.1 | stays 5.1 |
| `afir` (convolution) with a stereo impulse response on 5.1 | the four other channels are lost |
| Loudness of one test file | -14.0 LUFS measured as 5.1, -15.7 LUFS measured on its stereo downmix |

## Step 1: a player that takes surround gets it

**Declaring.** A protocol gets a property next to `channel_layout`
(`models/player_protocol.py:116,208`): the layout it wants its stream in, `None` for stereo.
`channel_layout` keeps describing the speakers (a Sonos home theater set reports 5.1 there and
still takes a stereo stream), so the two are separate. Step 1 accepts 5.1 and 7.1. No stored
setting and no models change.

**One helper answers "which layout does this player take now".** The format choices and the
buffer both ask it, so they cannot disagree. In step 1 it returns the declared layout for a
player that plays its own queue. A universal group declares none, and its route serves every
member a stereo format of its own (`providers/universal_group/constants.py:42`, verified), so its
members get stereo without a rule. A protocol that syncs natively (`PlayerFeature.SET_MEMBERS`)
gets its layout honoured from step 4 on; until then it gets stereo. That keeps a stream from
going on in six channels after a stereo member joins.

**Folding down keeps today's level.** FFmpeg scales its downmix only when the output is an
integer format (measured). Today every fold happens at decode into an integer format; with a wide
buffer it moves to where the buffer is read, often into float. So every fold to a narrower
layout uses explicit clip-safe coefficients (`aresample=ochl=…:rematrix_maxval=1`), in
`_get_channel_conform_filter` (`helpers/ffmpeg.py:894`). This also repairs a level error that
exists today for multi-channel DSD, which already folds in float (`helpers/audio.py:826-832`).

**The audio buffer.** The buffer of a track holds the source's channels in the nearest width (a
quad file in a 5.1 buffer), but never more than the player of its queue takes. A buffer is always
made for a queue item (`controllers/streams/audio.py:4426` is the only caller of
`AudioBuffer.get_buffer`), so the player is known. For every other queue the buffer stays stereo:
memory and behaviour do not change there.

- A buffer is reused through `streamdetails.buffer` when it is valid for the position
  (`controllers/streams/audio_buffer.py:592-631`). The width it was made for joins that check, so
  a track preloaded in one width is not played in another.
- A 5.1 buffer is three times a stereo one. The byte bound of DSD buffers
  (`audio_buffer.py:1069`, `DSD_BUFFER_MAX_BYTES`) also applies to a wide buffer. On the balanced
  setting that keeps about 155 seconds of 5.1 at 48 kHz and about 55 seconds at 96 kHz.

**The format of an item and of the flow stream.** `select_pcm_format`, `select_flow_pcm_format`
and `_select_audio_source_pcm_format` (`controllers/streams/audio.py:1584,1637,6219`) give a
surround player its own layout, for every source. A stereo track is widened where it is read from
the buffer (`controllers/streams/audio.py:2055`, the one place every playback read goes
through), and a mixed queue never changes format. When the output codec cannot carry the layout
(MP3, AAC), these return stereo, so nothing is folded inside an encoder.

**Crossfade, smart fades and overlays.** They mix in the format of the stream, and the filters
they use keep the channels (measured, except `rubberband`, which the test machine's FFmpeg
lacks). The analysis for smart fades averages all channels to mono
(`providers/smart_fades/helpers.py:150`). An overlay no longer forces stereo for a surround
player (`controllers/streams/audio.py:1634`); it lands on the front pair, a mono one duplicated
to stereo first. The processor load of a crossfade is about three times that of stereo.

**The output to the player.** `get_output_format` (`controllers/streams/audio.py:1579`) returns
the channel count of the player's layout.

- FLAC, WAV and raw PCM carry the layout. With MP3 and AAC the player gets stereo.
- The output channels setting (stereo, left, right, mono; `constants.py:561`) is not offered for
  a surround player, and a value stored earlier is ignored: `get_player_output_plan`
  (`controllers/streams/audio.py:1457-1486`) would fold the stream with it otherwise.
- The two shortcuts that write a WAV header in Python (`controllers/streams/controller.py:1289`
  and `:2326`, header at `helpers/audio.py:270`) have no channel mask. A surround stream goes
  through FFmpeg, which writes the extensible header.
- A provider that pulls PCM (`streams.get_stream`, `controllers/streams/controller.py:1910`) asks
  for its layout in the format it passes, as it passes rate and depth today.
- The announcement route builds its format from the URL (`controller.py:1831`); for a surround
  player it takes the layout, with the announcement on the front pair. A live announcement keeps
  the one or two channels its sender gave it (`controllers/streams/live_announcements.py:67`).

**DSP on a surround stream** (`helpers/dsp.py`).
- The preamp per channel builds `pan=stereo|…` (`:107`), which would fold a surround stream back
  to stereo. It becomes a pan in the layout of the stream that leaves the other channels alone.
- A band of the equaliser set to front left or right keeps working: FFmpeg addresses the channel
  by name (`:111`).
- Balance, stereo width and crossfeed already apply to a two-channel stream only (`:197`, `:254`,
  `:263`). Convolution (`:271`) loses the other four channels (measured), so it is skipped too.
  The DSP page says so on these four filters.
- Gain, tone, high and low pass, bands for all channels, compressor and limiter run on every
  channel, the LFE included. A high pass at 80 Hz therefore empties the LFE channel. That is
  accepted until step 6 and said in the docs.

**Loudness and analysis.** Analysis gets its audio in four ways, all through
`_start_analysis_on_providers` (`controllers/streams/audio_analysis.py:483`, `:510`, `:1599`,
`:1914`). Today they disagree for a surround file: playback measures the stereo downmix, the
background scan all channels (`:1599-1603`). From step 1 every analysis provider gets the stereo
downmix, folded once in that shared place, and the background scan asks for stereo at decode.

- A stereo player keeps the level it has today. A surround player uses the same value and can
  come out a little louder than the target.
- The audio fingerprint lookup refuses more than two channels
  (`providers/acoustid_lookup/provider.py:359`); with the fold it keeps working.
- Values the background scan stored earlier stay as they are. The loudness provider can
  re-measure everything through its version number (`providers/loudness_analysis/provider.py:58`);
  that is not done.

**What a player reports.** A surround source on a player with the same layout can be reported as
bit perfect (`controllers/streams/audio_processing.py:845` compares channel counts). A stereo
source on a surround stream is not.

**What it leaves alone.** The pacing on the stream endpoints
(`controllers/streams/constants.py:102`), the filesystem-only background analysis
(`controllers/streams/audio_analysis.py:1480`) and the rule that decoded audio is never written
to disk. No stored data changes.

**For provider authors.** A surround player in `_demo_player_provider`, and a paragraph in
`DEVELOPMENT.md`: declare the layout, play the server's stream, expect one fixed format in FLAC
order for everything from the queue, with stereo sources on the front pair.

**What the community provider then does.** It declares its layout (5.1 or 7.1, from its own
setting) and plays the server's stream for every source, as it does for stereo today
(`resolve_stream_url`, decoded by its own FFmpeg). Its own file reading goes. Its channel map
setting stays, as the wiring of the sound card.

**Pull requests.** Three, in this order:
1. Folding down at one level in every format, and analysis on the stereo downmix. No player
   declares anything yet; a stereo source is untouched and a multi-channel DSD source gets the
   right level.
2. The core: the declaration, the helper, the buffer width, the format choices, the output, the
   DSP fixes. It does nothing until a provider declares a layout.
3. The demo surround player and the text for provider authors.

Size: large, most of it tests. Files: `controllers/streams/audio.py`, `audio_buffer.py`,
`audio_analysis.py`, `controller.py`, `helpers/audio.py`, `helpers/ffmpeg.py`, `helpers/dsp.py`,
`models/player_protocol.py`, `controllers/config/players.py`, the demo provider, and one line of
text on the DSP page in the frontend.

**Verification.** A test file with a different tone per channel goes through the item route, the
flow route and the PCM route; each tone is found on its own channel, for FLAC and PCM, with
crossfade on and off. The same file to a stereo player gives the level it gives today, in integer
and in float. A stereo source to a stereo player is unchanged byte for byte. In the lab: the demo
surround player fetches a 5.1 track and a stereo track in one queue.

## Step 2: the upmixer

A select "Stereo sources" on a surround player, a new stored value per protocol (new data, so no
migration step):

- **Front speakers only** (default): what step 1 does.
- **All speakers**: left and right also on the surrounds
  (`pan=5.1|FL=FL|FR=FR|FC=0*FL|LFE=0*FL|BL=FL|BR=FR`). No delay, no change to the signal.
- **Surround**: FFmpeg's `surround` filter, which moves what left and right share to the centre
  and what is out of phase to the surrounds.

Measured for `surround` (stereo to 5.1, at 48 and 96 kHz):
- It looks ahead half its window: 2048 samples, 43 ms at 48 kHz. With
  `apad=pad_len=2048,surround=chl_out=5.1:lfe=0,atrim=start_sample=2048` the output lines up with
  the input to the sample and keeps its full length, so a group stays in sync and a track keeps
  its last 43 ms.
- It runs at more than 100 times real time on one core of the test machine.
- It lowers the level by 1 to 3 dB, depending on the content.
- By default it copies the bass into the LFE channel while the speakers keep it. A receiver that
  does its own bass management then plays it twice. So the upmix runs with `lfe=0`: the LFE
  channel stays empty for a stereo source.
- A track that is read in two parts (after a seek, and where a smart fade hands over to the rest
  of the track) differs from one continuous run for about 43 ms around the cut. Accepted.

**Where it runs.** Where a stereo or mono item is read from the buffer into the wide format
(`controllers/streams/audio.py:2055`), after the volume normalisation of that item. Only there is
the channel count of the item known: the flow stream behind it carries stereo and surround tracks
in one format. The upmix takes the place of the plain widening for that read; FFmpeg's own
conversion runs ahead of the other filters (`helpers/ffmpeg.py:780-783`) and must not come first.
A source with three or more channels is never upmixed.

Size: small to medium. `controllers/streams/audio.py`, `controllers/config/players.py`, tests on
what each channel carries.

**Not in this step.** The upmixer of a receiver itself (Dolby Surround, DTS Neural:X) only works
on a two-channel signal, so it never engages on a fixed 5.1 stream. A fourth choice "send stereo
as stereo" would need a stream that changes format between tracks, which costs gapless playback
and crossfade across such a change. It is left out until someone asks.

## Step 3: players that fetch the stream themselves

What can feed an AV receiver, checked in source code and manuals:

| Route | Verdict |
|---|---|
| The receiver's own network input | Stereo. Denon, Marantz and Yamaha document two channels for FLAC and WAV; Sonos documents mono or stereo files |
| squeezelite | Stereo from end to end (`squeezelite.h:450`, `BYTES_PER_FRAME 8`); a 5.1 FLAC plays its first two channels |
| AirPlay, also to an Apple TV | Stereo |
| Chromecast | Google documents surround only as AC-3 or E-AC-3 passthrough, which is lossy |
| A small computer on an HDMI input, with MPD | Works once Music Assistant sends 6-channel WAV: MPD takes up to 8 channels and puts FLAC order into the order of the sound device. No sync with other rooms |
| The same, with a software DLNA renderer (Kodi, upmpdcli) | Expected to work, not verified |
| The same, with the Local Audio app (sendspin-cpp-cli) | Step 5 |
| The same, with Snapcast | Possible in the library, but every client of a stream gets the same format |

So this step adds a setting "Surround output" (off, 5.1, 7.1; off by default) to a player whose
provider allows it: MPD and DLNA first, another provider when someone shows a device that plays
it. The provider adds the setting to its own config entries and its layout property reads it. A
new stored value, so no migration step.

- MPD is offered MP3, AAC and WAV (`providers/mpd/constants.py:14`: FLAC does not work there for
  an endless stream), so surround needs WAV. The settings page says so.
- To verify on a real MPD: that it reads a WAV with the extensible header from an endless HTTP
  stream, and the channel order on an HDMI output.

Size: small per provider.

## Step 4: Sendspin sessions and combined surround sets

Builds on the combined sets of the multi-channel players epic (#484: the stereo pair
music-assistant/server#7954 and the subwoofer PRs music-assistant/server#7999 and #8000). It
starts when those have landed.

Verified today:
- The session is stereo (`providers/sendspin/playback.py:1554-1564`).
- A member that needs DSP or a channel gets a pipeline of its own, with the session format going
  in and coming out (`:1442-1488`, `:1674-1680`); the others share one.
- A speaker of a set gets its channel as a pan of the stereo signal
  (`controllers/streams/audio.py:1458-1486`); the mapping knows front left and front right
  (`:277`), and the layouts of a set are stereo only
  (`providers/sendspin/multi_channel.py:56`).
- Nothing is missing in the Sendspin spec or in aiosendspin: every client already gets a stream
  of its own, and each speaker still receives mono or stereo.

What changes:
- **The session takes the width of its widest member.** A combined 5.1 set takes 5.1; a session
  without such a member stays stereo and is not touched.
- **Every member converts from the session.** A stereo member gets the downmix (shared by all
  stereo members without DSP), a speaker of a set gets its own channel. What goes over the wire
  per member stays stereo, so aiosendspin does not change. The pipeline of a member, its pan and
  the history it keeps are sized by the session format today (`playback.py:1278-1290`,
  `:1458-1464`, `:1674-1680`); they take the member's own format.
- **Members through a bridge** (AirPlay, Cast, squeezelite: `providers/sendspin/bridge_role.py:41`,
  two channels) are stereo members.
- **A channel for every speaker.** The mapping and the pan cover every `SpeakerChannel`. The
  downmix that runs ahead of the other filters (`helpers/ffmpeg.py:782`) must not run when a pan
  picks a channel.
- **A set with fewer speakers than the source has channels** (4.0, 3.0, 2.1): the session is
  folded into the layout of the set first, then each speaker takes its channel.
- **More layouts to combine**: the table of layouts and the compose dialog grow with centre,
  surround and side speakers.
- **Stereo sources** follow the select of step 2, set on the combined player.
- **Groups**: a protocol that syncs natively now gets its layout honoured, and a change of members
  restarts the stream when the width changes. The universal group gets the same rule: a wide
  session, a conversion per member.

To decide with the maintainer at the start of this step: what the other speakers play when one
speaker of a surround set is away, and how the LFE channel and the crossover of the subwoofer PRs
combine (the LFE channel is played 10 dB louder than the others by convention).

## Step 5: a Sendspin player that plays surround itself

For a computer with a surround sound card, or on an HDMI input of a receiver, in sync with the
rest of the house. This is the Local Audio app.

- **The spec** has no channel order (`roles/player/v1.md:13-32` at `2cfe197`). Sendspin/spec#350
  defines the FLAC order and one to eight channels; it is open and its fixed order is disputed.
- **aiosendspin** already encodes 6 and 8 channels in PCM and FLAC, in the same order
  (`audio/format.py:34-44`, `server/roles/player/capabilities.py`). Opus is mono or stereo.
- **The client** advertises stereo only and sets no channel map
  (`Sendspin/sendspin-cpp-cli` `src/supported_formats.cpp:50-59`); the Local Audio app's README
  says the channel count must be two. ALSA orders 5.1 differently from FLAC, so without a map
  the centre and LFE swap places with the surrounds.
- **Music Assistant**: a client that advertises a 6- or 8-channel format declares that layout
  (step 1), and its member pipeline of step 4 hands over the full width. Whether aiosendspin
  takes a different format per member is to verify.
- **Until then**: a device that presents one Sendspin client per output of its sound card, with
  those clients combined into a surround set of step 4, plays surround without any of the above.
  Roundabout, but a way to test.

## Step 6: DSP per channel

Level, distance and equaliser per centre, surround and subwoofer, bass management, and
convolution with an impulse response per channel. The DSP model knows three channels today: all,
front left, front right (models `dsp.py:14-24`). An unknown channel name is read as "all
channels" there, so a client with an older models version would apply a centre-only setting to
every channel. This step touches stored DSP settings; how existing ones are carried over is
decided with the maintainer when the step starts.

## Research: Atmos from Tidal

What Tidal delivers for an Atmos track, and whether using it fits the usage policy. A finding for
the maintainer, not a build step. Until then `providers/tidal/streaming.py:193` stays as it is.

## Kept open

- Three widths (stereo, 5.1, 7.1) cover every layout of `ChannelLayout`. Height channels do not
  fit in eight channels; they would need a layout on `AudioFormat`, which nothing here rules out.
- A layout is a `ChannelLayout` and a channel is a `SpeakerChannel`. Never a filter string, and no
  API or stored value says "six".
- The layout a player wants its stream in is separate from the layout of its speakers.
- A stream that follows the channel count of the source (step 2, "not in this step") stays
  possible: it is a choice per player on top of the fixed layout.

## Verification

- Step 1 and 2: unit tests on the format choices, the fold and the DSP filters at 5.1 and 7.1,
  and the per-channel tone test through every route, with a stereo player as the negative
  control.
- Step 3: a real MPD on an HDMI output, tested by the maintainer on his own receiver.
- Step 4: the end-to-end lab with virtual Sendspin clients that record what they get, as for the
  stereo pair: a tone per channel, each found at its own speaker, alone and in a group with a
  stereo player.
- Step 5: the Local Audio app on a device with a 6-channel output.
