# Smart fades for every music style: technical plan

Status: proposal 2026-10-10. Owner: Marcel van der Veldt. This document is the technical
companion of the "Smart fades that suit every music style" epic (#286) on the project board. It is
written so it can be fed to an agent for implementation; every claim marked "verified" was
checked in the referenced code (server `origin/dev` at `24dbf8c2c`, paths relative to
`music_assistant/controllers/streams/smart_fades/`) or reproduced from the replay data.

## Summary

The smart fades planner is built around one idea: a good transition is a beatmatched DJ blend.
When a pair of tracks can't be beatmatched, it falls back to a "quick fade" of 1 to 4 bars whose
length depends only on the tempo gap. In a mixed queue that is four transitions out of five, so
listeners hear a 2 to 4 second fade almost everywhere, also where a long fade would sound fine.

Players that handle mixed styles well (Plexamp's Sweet Fades, MPD's MixRamp, Liquidsoap's
autocue, radio automation) don't look at tempo at all. They start the next track where the
outgoing track has gone quiet and let the recorded fade do the work. Spotify's DJ-style
transitions beatmatch, but its research system only ever mixed pairs within 5 BPM and its Mix
feature offers 2 bars when tempos don't match.

The plan keeps the beatmatched blend for rhythmic, tempo-compatible pairs and makes a loudness
segue the default for everything else. A drum clash check joins the existing vocal clash check,
so the length of a fade follows what would actually collide. Two short dressed transitions
(filter out, echo out) cover two beats at very different tempos, and a last step adds phrase
boundaries and mixing inside the loud part of a song. Everything is computed from stored
analysis; no stored data or settings change.

## Decisions

Taken with the maintainer on 2026-10-10:

- Direction: length follows a clash check (vocals, drums, loudness), not the tempo tier; the
  planner gets a set of transition styles scored by the same policies.
- The loudness segue is the default for every pair that can't or shouldn't be beatmatched. The
  beatmatched blend stays for rhythmic pairs with compatible tempos.
- Tempo stretching stays small (the current 8 % limit) and only runs where it helps. Stretching
  the incoming track to the outgoing tempo over larger gaps (Spotify Mix "lock and release") is
  not part of this epic.
- Styles in scope: long fade where one side has no beat (part of the segue), filter out and echo
  out, phrase-aware placement and mixing in the loud part.
- Results must hold across music styles, measured on a mixed-styles playlist.
- Three small bugs ship first in their own PR (sub-issue 1).

## Requirements

- A pair that can't be beatmatched gets a fade as long as its quiet outro and quiet intro allow,
  up to about 15 seconds, unless two vocals or two beats would play on top of each other.
- One-sided vocals never shorten a fade.
- A quiet but audible outro gets a segue, not the standard fallback.
- Two loud beats at very different tempos get a short transition on a phrase boundary, dressed
  with a filter sweep or an echo, never a long overlap.
- Beatmatched blends sound as they do today; stretching never exceeds 8 % and never runs on
  material without a beat.
- Every transition logs one DEBUG line with its style and reason.
- Planning stays pure over the analysis rows (no audio bytes), cheap enough to run per boundary.
- Nothing changes about how audio is fetched, kept or exposed (usage policy); no decoded audio
  is written anywhere.

## Measurements: the current planner

Replay of `SmartCrossFadePlanner.plan()` over stored analysis (smart_fades analysis version 3),
called with the buffer production would pass (`min(45, duration / 2)`), 3,000 random ordered
pairs from different albums, seed 20261010. Two corpora: a home library (357 tracks: local files,
Qobuz, Tidal, Spotify) and the dev test server (1,147 tracks, 86 % with a genre bucket). The
script and per-pair CSVs are listed under "Tooling".

| | home | dev |
|---|---|---|
| QUICK_FADE tier | 81 % | 79 % |
| shipped overlap under 8 s | 80 % | 79 % |
| median overlap | 3.5 s | 3.8 s |
| pairs within 8 % BPM | 21 % | 23 % |
| not applicable | 2.6 % | 2.5 % |

Why quick fades happen (first trigger, home corpus): BPM gap above 8 % 68 %, different meter
12 %, irregular downbeats 12 %, fewer than 8 anchored downbeats 8 %. Folding half or double time
brings only 5 % of the over-8 % pairs within 8 %.

Vocals are not the cause. Of the short fades where an 8-bar fade ending at the same point would
have had no vocal overlap at all (45 % of all short fades), 94 % were short because of the tier.
By vocal class over an 8-bar window (duty above 0.10 = sings):

| class | pairs | under 8 s | caused by a vocal rejection |
|---|---|---|---|
| both sing | 1,450 | 87 % | 237 |
| outgoing only | 800 | 73 % | 0 |
| incoming only | 469 | 76 % | 0 (34 by the audible-trim rule) |
| neither | 203 | 75 % | 0 (20 by the audible-trim rule) |

Drums decide whether a longer fade would work. For the quick fade pairs, over 8 bars: 36 %
have an outro or intro without a kick (home; 42 % on dev), 33 % would put two kicks on top of
each other for more than 2 gain-weighted bars (29 % on dev), the rest overlap less. This barely
changes with the tempo gap.

Music style makes no difference (dev corpus, outgoing / incoming side, share under 8 s): rock
78 / 77 %, soul/r&b/funk 84 / 82 %, pop 80 / 88 %, house/electronic 77 / 76 %, country/folk
78 / 80 %, latin/reggae/world 78 / 86 %, ambient 77 / 84 %. House to house does better (65 %
under 8 s) only because 44 % of those pairs are within 8 % BPM, against about 20 % elsewhere.

With 20 seconds of room instead of 45 (common on a source that streams one track at a time),
half of all outgoing tracks become "not applicable", almost all of them through the mix-out
check below, not through silence.

## How it works today

Verified at `24dbf8c2c`.

**Tier.** `choose_tier()` (`planner/context.py:583`) returns QUICK_FADE when the meters differ,
when `_tail_is_blendable()` fails (`:609`, fewer than 8 anchored downbeats or interval std
≥ 0.1 s) or when the BPM gap exceeds `TIME_STRETCH_BPM_PERCENTAGE_THRESHOLD` (`:66`, 8 %).
Otherwise FULL_BLEND (key compatible, RMS data, 4/4) or TEMPO_BLEND.

**Length.** `bars_ladder()` (`planner/candidates.py`) gives FULL_BLEND 8 bars (16 when both decks
are near-instrumental), TEMPO_BLEND 8, QUICK_FADE by `_QUICK_FADE_LADDER` (`:74`, 4 bars up to
12 %, 2 up to 20 %, 1 beyond). The cross-meter branch (`:137`, `((0.0, 2),)`) only matches a 0 %
gap, so cross-meter pairs get 1 bar (fixed in sub-issue 1). Every generator walks `RUNG_LADDER`
(`:76`) downwards from that top rung.

**Selection.** Generators (`default_generators()`, `:385`) emit specs; `CandidateFactory.build()`
times them; policies (`planner/policies.py`) reject or penalise: `VocalCollisionPolicy` (`:65`),
`VocalTruncationPolicy`, `AudibleTrimPolicy` (`:116`, rejects a short fade that trims more
audible tail than its own length), `DeadAirPolicy`, `OverlapPreferencePolicy` (`:179`, 10 per rung
below the top, 15 per tier step), `AnchorAlignmentPolicy`. When all are rejected the planner
runs a rescue pass, then a plain fallback crossfade, then the 0.4 to 1.0 s emergency handoff
(`planner/planner.py:130-133`).

**The one long non-beatmatched option.** `LazyOverlayGenerator` (`planner/candidates.py:360`)
emits a 16 s unphrased equal-power overlay, but only for QUICK_FADE pairs within 8 % BPM with
both decks at or under 10 % vocal duty. It shipped on 0.3 to 0.4 % of pairs.

**Not applicable.** `_cue_outgoing_tail()` takes `min(audible end, mix-out or kick anchor)` and
raises when it is under `MIN_EFFECTIVE_FADE_BUFFER` (`helpers.py:16`, 8 s, counted from the
buffer start) with "outgoing tail is mostly silent" (`planner/context.py:380`).
`detect_mix_out_point()` (`helpers.py:249`) returns 0.0 when the bar-smoothed tail stays below
`MIX_OUT_ENERGY_FRACTION` (0.70) of the track's median level (`:296`). The audible-end detector
(`detect_effective_audio_end`, `helpers.py:23`, floor 5 % of median) is sound: it drops a median
4 s, the end of a mastered fade. The small value in the message is the mix-out point, so a quiet
musical outro (Bohemian Rhapsody: 14.5 s of vocals in the tail; Faithless We Come 1: level
until the last second) is reported as "0.0s/1.0s audible" and gets the standard 8 s fade at the
very end.

**Render.** `TransitionRenderer` (`renderer.py`) chains `FadeOutTrimFilter`, shelf and peak EQ
driven by `asendcmd`, `GradualTimeStretchFilter` (rubberband on the outgoing branch only),
`FadeInTrimFilter` and `StreamingCrossfadeFilter` (afade + adelay + amix, positioned by a
pre-roll, configurable curves, `nofade` used for mastered fades by `_choose_fadeout_curve`,
`planner/assembly.py:791`). Quick fades are pure volume fades (`_choose_eq`, `assembly.py:150`).

## What other players do

Evidence levels: code, official docs, developer posts, reviews.

- **Spotify research** (Bittner et al., ISMIR 2017, Spotify; patent US10803118B2). Fixed
  transition length per pair. It searches every downbeat pair in the last 25 % of the outgoing
  and first 20 % of the incoming track for the lowest cost: timbre and chroma distance, both
  tracks loud, not both singing, ending on a section boundary or drop. Tempo is ramped beat by
  beat. The listening test only used pairs within 5 BPM; the playlist is reordered by tempo, key
  and timbre first. Curators rated 64 % good; 15 % were bad because beats did not align, 0 %
  because of a key clash. (paper, patent)
- **Spotify Mix** (2025). Volume, EQ and filter lanes; low-pass and high-pass are the only
  effects (official support page). 2, 4 or 8 bars; mismatched BPMs get a warning and 2 bars;
  matching pairs are stretched to the outgoing tempo and released afterwards (reviews and
  community threads only).
- **Plexamp Sweet Fades.** No gain curves: the next track starts when the outgoing one drops
  below a fixed level, both play at full volume, overlap capped at 15 s, albums gapless.
  ("there's no actual crossfading going on, it's just finding optimal overlap points", Plex
  developer on the Plex forum.) A third-party client reads Plex's ramps as MixRamp data.
- **MPD MixRamp** (code, `src/player/CrossFade.cxx`). Overlap = time the outgoing track spends
  below the threshold at its end plus the time the next track needs to reach it; no overlap when
  the threshold is never crossed.
- **Liquidsoap** (code, `cross.smart`): compares the end level of A with the start level of B;
  both quiet and similar = crossfade, one side louder = fade only the other side, both loud =
  no overlap. Its autocue starts the next track at the last point above -7 LU relative to the
  track's integrated loudness and caps the overlap at 6 s; Moonbase59's autocue (AzuraCast)
  uses -8 LU and re-scans 12 LU lower when the overlap would exceed 15 s.
- **Radio automation** (mAirList, RadioBOSS, ENCO, StationPlaylist). A segue marker at a level
  threshold (-14 to -30 dB); StationPlaylist Pro also uses a vocal-start marker.
- **djay Automix** offers fade, filter, EQ, echo and riser transitions, plus stem separation.

Shared scheme: overlap only what is already quiet, thresholds relative to the track, length from
the material with a cap, fades only on a side that is still loud, both loud means no overlap.
None of these players checks tempo. Spotify Mix and djay show that filter sweeps and echo are
the usual way to dress a short transition between mismatched beats.

## Design

### Families and styles

A `TransitionStyle` on the candidate spec says how a candidate is timed and rendered:

| family | style | when |
|---|---|---|
| beatmatched | `BLEND` (today's FULL/TEMPO) | both rhythmic, tier FULL or TEMPO |
| segue | `SEGUE` | default for every other pair |
| dressed short | `FILTER_OUT`, `ECHO_OUT`, `CUT` | segue rejected because both ends are loud and the beats or vocals clash |

`TransitionTier` stays as the beat-compatibility verdict (it still gates beatmatching, tempo
ramps and EQ). `TransitionStrategy` keeps its meaning (how the final overlap was decided);
`LAZY_OVERLAY` folds into `SEGUE` once sub-issue 4 lands. Every style is a generator plus a
factory path; the selector, policies, rescue pass and fallbacks stay as they are, so a style
that never wins changes nothing.

### Clash checks

- **Vocal** (exists): `collision_seconds` and the gain-weighted integral, limits 2.0 s and 0.35.
- **Drums** (new, sub-issue 3): per-bar kick presence for both decks from the stored low band
  (20 to 120 Hz) in the `BandProfile`, a bar counting as kick when its low power reaches 0.5 of
  the track's reference (the planner's existing `_LOW_ANCHOR_BAR_FRACTION`). The candidate
  metric `rhythm_clash_bars` integrates "both kick" under the fade's simultaneous gain, the same
  weighting as the vocal check. A `BLEND` candidate with tempo lock has zero clash. Start
  limit: reject above 2 weighted bars, quadratic penalty below.
- **Loudness** (new, sub-issue 4): per deck, the quiet tail and quiet head measured on the
  bar-smoothed energy envelope relative to the track's sustained level (the median of active
  bins, as `sustained_energy_floor` already computes).

### Segue (sub-issue 4)

1. Outgoing: the segue point is the last moment the smoothed energy is at or above `T_out` of
   the sustained level (start value -8 dB, tuned by replay and listening); the quiet tail runs
   from there to the audible end. Snap to a downbeat when the grid is reliable, else leave it.
2. Incoming: the quiet head runs from the entry to the first moment the energy reaches `T_in`
   (start value -8 dB).
3. Overlap = quiet tail + quiet head, at least 2 s, capped at 15 s. A tail longer than the cap
   moves the segue point later (the next track starts later), as Moonbase59's long-tail rule.
4. Curves: `nofade` on a side that is already quiet over the overlap, equal-power (`qsin`) on a
   side that is still loud.
5. Clash: shrink the overlap until the vocal and drum checks pass; when both ends are loud and
   it can't shrink below the clash, reject and let the dressed short styles compete.
6. A quiet musical outro is segue material, so the "mix-out too early" branch of
   `_cue_outgoing_tail` plans a segue instead of raising; only a silent tail raises.

The segue must also fit the room the boundary has (`buffer_duration`), which is often 10 to
28 s on one-slot realtime sources.

### Stretch only where it helps (sub-issue 5)

`_choose_tempo_ramp` runs only when both decks have kick bars in the stretch window and the
overlap. A beatless side blends unstretched (and usually wins as a segue anyway). The 8 % limit
stays.

### Dressed short transitions (sub-issue 6)

Placed on the outgoing downbeat nearest the energy anchor; sub-issue 7 moves it to a detected
phrase end.

- `FILTER_OUT`: 2 to 4 outgoing bars. A high-pass on the outgoing branch sweeps from 20 Hz to
  about 600 Hz with a volume fade; the incoming track enters on its downbeat at full range. New
  `HighPassSweepFilter`: ffmpeg `highpass` supports a runtime `frequency` command (verified with
  `ffmpeg -h filter=highpass` on ffmpeg 9), so it uses the same `asendcmd` pattern as
  `ShelfFilter`.
- `ECHO_OUT`: at the phrase end the outgoing dry signal stops within one beat and its last beat
  repeats at the outgoing tempo with decaying taps (`aecho` with delays at 1, 2, 3, 4 beats and
  decays 0.5, 0.25, 0.12, 0.06; `aecho` has no feedback, verified option list) over the first
  1 to 2 incoming bars. New `EchoOutFilter` on the outgoing branch: split, gate the dry path,
  `atrim` the last beat, delay, echo, `apad`, mix.
- `CUT`: today's 1-bar quick fade, kept as the last resort.

Choice: all three are candidates; the policies pick. Start bias: `ECHO_OUT` above 20 % tempo gap,
`FILTER_OUT` from 8 to 20 %, `CUT` when the incoming head starts with a strong energy step.

### Phrase-aware placement and the loud part (sub-issue 7)

Section boundaries from bar-level band energy novelty (`BandProfile.bar_power`), phrase units of
4 and 8 bars from each boundary. Generators add anchors on phrase ends and entries on phrase
starts. A new candidate kind may anchor inside the loud part of the outgoing track (before the
mix-out point) when its outro is long and repetitive and the incoming head is loud, scored by a
"both loud" term in the spirit of Spotify's cost; `AudibleTrimPolicy` stays the guard, with its
penalty relaxed only for a repetitive outro.

### Logging

One DEBUG line per transition from the planner: style, tier, overlap, the reason (for example
`segue: quiet tail 9.2s + head 1.5s, curves nofade/qsin, vocals out-only, kick in-only`). The
per-candidate scoreboard stays at VERBOSE.

## Tooling (sub-issue 2)

- Replay script: reads copies of `audio_analysis.db` and `library.db`, rebuilds
  `AudioAnalysisData` exactly as `SmartFadesMixer._load_analyses` does, loads the planner from a
  given checkout, runs random or same-album pairs at a given buffer, and writes a per-pair CSV
  plus a summary (tier, style, overlap, cause, vocal class, drum clash, music style bucket from
  track genres). Analysis data only, never audio. Open question: commit it under `scripts/` or
  keep it out of the repo.
- Mixed-styles playlist on the dev server ("Smart fades test: mixed styles", 38 tracks across
  jazz, classical, hip-hop, soul, house, ambient, rock, metal, pop, ballads, country, reggae,
  latin, funk, blues and waltz time). Listening rounds play it shuffled with
  `smart_fades_log_level` at VERBOSE; a small parser turns the log into a per-transition report.
- Listening A/B renders only from local files (usage policy).

Each sub-issue reports the replay table above before and after, and a listening round on the
mixed-styles playlist for the ones that change audio.

## Risks

- Thresholds. -8 dB relative and the 15 s cap come from other players; our energy envelope is
  peak-normalised RMS, not LUFS. Tune by replay and listening before calling it done.
- Vocal detector false positives shorten segues; the existing saturation abstain stays.
- Mastered fades: a segue over a mastered fade must not fade twice; `nofade` already handles
  this for blends and becomes the default for a quiet side.
- Echo render inside the streaming graph: the incoming input can still be arriving
  (`StreamingCrossfadeFilter` emits progressively); the echo must only touch the outgoing branch.
- Short rooms on realtime sources cap every style; the planner already receives the real buffer.
- BPM octave errors exist (a few tracks read at half or double tempo) but are not a driver.
- First plays have no analysis and keep the standard crossfade.

## Sub-issues

One PR each, in this order; 3 is folded into 4, 5 and 6 need 4, and 7 refines where 4 and 6
place a transition.

| # | issue | sections | size |
|---|---|---|---|
| 1 | #287 Server: small smart fades fixes | How it works today | tiny |
| 2 | #288 Tooling: replay and listening harness | Tooling, Logging | small |
| 3 | #289 Server: a clash check for drums (folded into #290) | Clash checks | - |
| 4 | #290 Server: segue for tracks that can't be beatmatched, with the drum clash check | Clash checks, Segue | large |
| 5 | #291 Server: stretch only where it helps | Stretch only where it helps | tiny |
| 6 | #292 Server: filter out and echo out | Dressed short transitions | medium |
| 7 | #293 Server: phrase-aware placement and mixing in the loud part | Phrase-aware placement | large |
| 8 | #294 Docs: transition styles | all | small |

## Not in this epic

Stretching the incoming track over larger tempo gaps, stem separation, riser or reverb effects,
per-genre rules, user-editable transitions, Smart Shuffle ordering by segue quality. No new
settings and no stored data changes, so no migration.

## Changes during implementation

- 2026-10-10, #287 (server#6833): the live log level fix covers every logger the streams
  controller derives (audio, ffmpeg, smart fades), not only the smart fades one. The cross-meter
  cap is `min(2, tempo ladder)`, so a cross-meter pair more than 20 % apart still gets 1 bar.
- 2026-10-10, #288 (server#6841): the replay tool lives in the repo as
  `scripts/smart_fades_replay.py`. It copies the databases with SQLite's backup API, so it can
  run against a live server, and stops on any planner error other than "not applicable". Random
  pairs only; same-album pairs were dropped because same-album tracks don't crossfade by default,
  and "same album" follows playback's rule (the track's own album). `--out-bucket` and
  `--in-bucket` stay for per-style runs.
- 2026-10-10, #288: the per-transition DEBUG line ("planned transition: tier=... trigger=...
  strategy=... source=... bars=... overlap=... bpm=...") replaces the three "shipping ..."
  lines. The quick fade trigger is checked in the order meter, tempo, beat grid, so tempo is
  named when both tempo and grid rule out a blend (167 of 208 grid cases were also more than 8 %
  apart); a blend context whose candidate re-anchors into a quick fade logs the beat grid.
- 2026-10-10, #289 folded into #290: a drum clash rejection without the segue would shorten
  today's 4-bar quick fades, so the metric and `RhythmClashPolicy` land together with the
  segue, where they decide how long a segue may run.
- Baseline after #287, dev corpus, random @45 s: 79.0 % of fades under 8 s, median 3.9 s;
  quick fade triggers over shipped quick fades: tempo 2056, meter 242, beat grid 71.
