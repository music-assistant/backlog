# An audio source knows who uses it: technical plan

Status: decided on 2026-11-16; step 1 is built as draft music-assistant/server#7800, steps 2 and
3 follow (see Decisions and Steps). Owner: Marcel van der Veldt. This document is the technical
companion of the story "Playback stalls after a queue change on a music service that allows one
stream at a time" (#481) on the project board.
Written against server `dev` at `077802313`; every `path:line` below is on that commit. It
follows from pull request music-assistant/server#7762, a first fix that was closed, and from what
its review rounds found (see the appendix). A throwaway prototype was used to check the claims
(see "What was checked").

Paths are relative to `music_assistant/`. The folder is left out for `audio.py`,
`audio_buffer.py` and `audio_analysis.py` (in `controllers/streams/`) and for
`stream_feeder.py`, `queue_loader.py` and `smart_shuffle.py` (in `controllers/player_queues/`).
A bare `controller.py` is the queue controller; the streams controller is always written
`streams/controller.py`.

## Summary

**Does it hold up?** Yes, with one precise limit. A source can know its *users* by itself: every
party that waits for its audio or reads it passes through two places in `audio_buffer.py`, so the
buffer registers them without any call site keeping a marker. What a source cannot know by itself
is whether its queue still *plans* to play it. That is read from the queue at the moment of the
decision, from the position it already keeps, with no new stored state.

**The rule.** A source is needed while someone uses it (waits for its audio or reads it), or
while its queue plays its track now or next. A source that still fills and is not needed is
stopped: always when its track is on no queue any more, and when its track only moved down the
queue as soon as its provider has no free slot.

**When it is checked.** One function, three moments: when another track asks the provider for a
slot and none is free, when the last user of a source leaves, and when the items of a queue
change.

**What it replaces.** All five inferences of pull request 7762 (open stream requests, the flow
marker, the pending fade, the buffer identity check, the release at the end of a request). On
`dev` it adds the user set and deletes little; its value there is three gaps that close without
special cases (see section 3.6).

**Steps.** Three pull requests. The first one alone ends the incident in all four forms. It only
acts where a provider has no free slot, or where the last user leaves a source nothing can reach
any more. The second makes the release prompt at the edit. The third deletes what the rule made
redundant.

**Choice.** The proposal (c), cut so that its first step has the reach of the full design and a
blast radius close to that of the narrow variant.

## Decisions

Taken with the maintainer on 2026-11-16:

- The proposal of section 3 is built, in the three steps of section 6, in place of pull request
  7762. Each step ships on its own and spends a night in the nightly build before the next.
- A prepared track that only moved down the queue gives way only when its provider has no free
  slot (open question 1, choice a).
- A prepared next track keeps its source when a speaker asks for another track by itself (open
  question 2, choice a).
- The 15 second wait for a slot stays as it is for now; it is its own small change after step 1
  (open question 3).
- Steps 1 and 2 ship apart (open question 4).
- The new next track may start cold after an edit (open question 5, choice a).
- A "play next" in flow mode that lands after the flow selected its next track is not part of
  this work (open question 6).

Step 1 as built differs from the estimate below in two ways. It is 283 production lines rather
than 150, and four streams (the buffer's processed stream, the item stream, the fading item
stream when it is cut off, and a mix that is abandoned before it is handed over) now close what
they read from when they end: a reader only leaves its buffer at once when its stream is closed,
not when the speaker disconnects.

## What was checked

* The four incident forms on today's `dev`: the incident tests of pull request 7762 were run
  unchanged against `077802313`. Six fail with `ProviderStreamLimitError` (replace next, play
  next, remove, move, repeat wrap, and the fade case); 17 pass. Remove and move were "from code
  reading" in the appendix; they are now confirmed.
* A prototype of the rule (178 added lines in four files, not kept) against the same file: all
  incident cases pass, as do the two flow cases of rounds 1 and 2. Four tests fail by design:
  they simulate "in use" by setting a marker by hand (`requested_item_ids` mocked, an entry put
  into `_crossfade_pending`) instead of having a reader.
* Ten probes with real readers for findings 3, 5, 6, 7 and 8, the window "next lands while the
  preparation fetches details", and the need-time check on its own. All pass. Each part of the
  prototype was then switched off in turn; the probes that depend on it fail, so none of them is
  vacuous. With the hook in the queue edit switched off, every incident test still passes.
* 620 existing tests in 31 stream and queue test files against the prototype: 615 pass. The 5
  that fail use a bare namespace as a buffer. Before the prototype tolerated stand-ins, at least
  57 tests in five files failed for the same reason (a mocked provider or buffer without the new
  member); each of those files builds its stand-ins in one helper.

Not in the prototype: the "queue has no session" half of the rule, the two extras of 3.7 and the
deletions of step 3. Where something is stated that was not run, it is marked as an assumption or as
"from code reading".

## 1. Users of a source

A source is an `AudioBuffer` plus its producer task. The producer iterates
`StreamsAudio.get_media_stream`, which holds the provider's slot for as long as it runs
(`audio.py:829-837`, started by `audio_buffer.py:359-405`). So "give the slot
back" always means: the producer ends, by itself or through `AudioBuffer.clear()`
(`audio_buffer.py:407-444`).

Today the buffer knows nothing about its users. It has a last-access time
(`audio_buffer.py:139`), a read position and the cancel callbacks of the analysis reader.

| # | User | Starts needing the source | Stops | How the code knows today | Needs it but does not read |
|---|---|---|---|---|---|
| 1 | Start of an item (`_play_index` → `_load_item(is_start=True)`, `queue_loader.py:496-501`) | the buffer is created inside the play action | the item is done or the session ends | `index_in_buffer = index` (`controller.py:3068`), the session stamp on the details (`audio.py:4098-4102`) | from "buffer ready" until the player's request arrives: the play command and the player connecting |
| 2 | Preparation of the next track (`stream_feeder.py:56-177`; triggered at `audio.py:1932-1948`, `stream_feeder.py:179-198`, `audio.py:2299`, `audio.py:2803`) | `get_audio_buffer(reason="prepare_next")` (`stream_feeder.py:130`) | the next track's reader takes over | while the task runs: its task id and `next_item_id_preparing`. After the task: nothing but the position | the whole time between the end of the task and the first read. **This is the holder of the incident.** |
| 3 | A speaker's stream request (`streams/controller.py:910-1410`) | its registration (`:1002-1003`) | its `finally` (`:1362-1385`) | `_open_item_streams`, `_ended_item_streams`, `last_served_item_id` | the sweep (`:1013`), stream details, format selection, the wait for a fade (up to 30 s, `audio.py:5012-5052`), the wait for a slot (up to 15 s), the encoder start. A HEAD request never reads |
| 4 | A direct PCM consumer (`StreamsController.get_stream`, `streams/controller.py:1826-1977`): AirPlay, Sendspin, Snapcast, Squeezelite multi-client, the universal group | the provider iterates the generator | it closes it | for sources: not at all. `_open_item_streams` only holds HTTP requests (`:1003`, `:1684`) | generator created, not yet iterated |
| 5 | The flow generator (`audio.py:2542-3314`); more than one per session is possible (`audio.py:2606-2612`) | per track: `load_next_queue_item` returns (`:2672`) | the track's last chunk | `index_in_buffer`, raised at the track's *first chunk* (`:2943`). Between selection and first chunk: nothing | selection → first read: tail play-out (`:2803-2820`), the mixer build (`:2851`), the prefetch handover (`:2909`, up to 5 s) |
| 6 | The flow prefetcher (`audio.py:422-631`) | `ensure_started` (`:456-532`) | `take` hands its stream over (`:534-570`), or `close` | only inside the prefetcher | after it collected its target it holds the open stream without reading |
| 7 | A crossfade outside flow mode (`audio.py:1991-2540`): the request of track A uses track B | `load_next_queue_item` at A's end (`:2216`) | B's own request has played the rest of the mix | `index_in_buffer = B` right at selection (`:2225`), then `_crossfade_pending` (`:2265-2270`) and `_crossfade_handover` (`:2458-2460`) | selection → first pull of the mix; the handover between the two requests |
| 8 | Gapless handover: the player holds the next track's URL (`stream_feeder.py:329-410`) | `enqueue_next_media` | the player requests it (then it is user 3), or the queue hands over another one (`:303-327`) | `next_item_id_enqueued` | all of it; the source may not exist yet |
| 9 | Audio analysis (`audio_analysis.py:396-448`) and the visualizer tap (`providers/milkdrop_visualizer/tap.py:404-417`) | the buffer is created | the buffer is cleared (cancel callback, `audio_buffer.py:426-432`) | cancel callbacks | never keeps a source, by design (`audio_buffer.py:281-283`) |
| 10 | A transferred queue (`controller.py:1396-1557`) | the target's own start | as user 1 | `item.queue_id`, session stamps | the same `QueueItem` objects go to the target (`:1531-1534`) while the source queue's stop and clear still release their audio in tasks (`:2581`, `:2837`) |
| 11 | A group | one consumer: the universal group opens one stream for its queue (`providers/universal_group/player.py:325`), a sync group plays the leader's | as user 3, 4 or 5 | as those | members have no source of their own |
| 12 | Announcements | no source buffer (streams README, "Stream Types") | | | they matter only through the stop and resume of the queue |
| 13 | A paused queue | its session keeps its buffers | the pause watcher, or `release_paused_stream_slot` (`stream_feeder.py:268-301`) | queue and player state | the whole pause |
| 14 | Slot takers without a buffer: the format probe (`audio_buffer.py:1012-1066`), the preview stream (`streams/controller.py:1995-2036`), a provider that takes a slot itself (`providers/yandex_ynison/provider.py:866`) | | | the semaphore only | |

Two providers read the buffer's identity from inside their own source
(`providers/bbc_sounds/streaming.py:88-104`, `providers/bookorbit/__init__.py:1044-1045`). They
depend on "detached" and "cancelled" keeping their meaning; the proposal keeps both.

What the table shows: five kinds of reader (3, 4, 5, 6, 7), and each is known through a different
marker or through none. All five read through `get_queue_item_stream` (`audio.py:1677`) and end
in `AudioBuffer.get_raw_stream` (`audio_buffer.py:244-275`).

## 2. Release paths today

| # | Where | Trigger | What it trusts |
|---|---|---|---|
| R1 | the producer ends (`audio.py:835`) | the source is done or fails | nothing |
| R2 | `get_buffer`, existing buffer not valid (`audio_buffer.py:484-507`) | a seek the buffer cannot serve, an error | `is_valid`, "provider has no free slot", or "last access more than 30 s ago". Otherwise the old buffer is left for "an active consumer" and the inactivity timeout |
| R3 | `_clear_failed_buffer` (`audio_buffer.py:591-602`) | the buffer does not become ready, or its producer failed | the producer error |
| R4 | `_monitor_inactivity` (`audio_buffer.py:803-826`) | 300 s without access | `_last_access_time`, which `is_valid()` also resets (`:225`) |
| R5 | `_do_prepare`, after the buffer is ready (`stream_feeder.py:140-152`) | the prepared item left the queue, or the session ended, *while the task ran* | `get_item`, `session_id` |
| R6 | `_do_prepare`, cancelled (`stream_feeder.py:155-165`) | another preparation replaces it, or a stop cancels it, *while the task ran* | the cancellation |
| R7 | `_cleanup_stale_queue_buffers` (`stream_feeder.py:466-507`) | a track's first chunk went out | list position: everything at `index - 2` and before |
| R8 | `_cleanup_queue_audio_data` (`stream_feeder.py:509-556`) | stop and failed start (`controller.py:2581`), clear (`:2837`), end of queue (`:989`), replace (`queue_loader.py:209`, `:1422`) | the queue's item list at that moment, the session stamp |
| R9 | `_release_queue_audio` (`controller.py:2602-2613`) | the player is detached or removed | the item list |
| R10 | dynamic rebuild, discarded tail (`queue_loader.py:1438-1443`) | an enqueue on a dynamic queue | `committed_index`, `requested_item_ids` (`:1394-1401`) |
| R11 | `_abort_superseded_source_buffers` (`queue_loader.py:1532-1568`) | the start of an item | list position (`index + 1`, not `get_next_item`), "still filling", "provider has a limit", for the successor "no free slot" |
| R12 | `release_failed_item_source` → `_abort_source_buffer` (`controller.py:1876-1890`, `queue_loader.py:1570-1611`) | a request ended and the player asked for a newer item (`streams/controller.py:1375-1385`); the sweep at every GET (`:1013`, `:2653-2687`); a speaker reports a failed item (`providers/sonos/player.py:910`) | the request history, and `get_item`: an item that left the queue is not found |
| R13 | `release_paused_stream_slot` (`stream_feeder.py:268-301`) | another queue needs the slot | `_held_stream_slots`, which infers "holds a slot" from "still filling and ready" (`stream_feeder.py:630-639`) |
| R14 | fade data: `clear_crossfade_handover` (`audio.py:3462-3474`), a claim from another session (`audio.py:5001-5009`), a stale handover at the next publish (`audio.py:2441-2443`) | stop, clear, a new start (`controller.py:3095`) | the queue id, the session id |

Not on the list: `update_items` (`controller.py:2063-2082`). It swaps the list and releases
nothing. R5 and R6 only work while the preparation task still runs; once the buffer is ready the
task is over and the capture goes on with no path that reaches it. That is the incident. Its
callers are `load` (`:2061`), move (`:885`, `:919`), delete (`:945`), transfer (`:1534`), the
skip of unplayable items (`:1829`), restore (`:2154`), clear (`:2838`), Smart Fades ordering
(`smart_shuffle.py:200`), and two plugins (`providers/party/__init__.py:774`, `:790`,
`providers/ai_radio/queue_dj.py:455`, `:557`, `:588`, `:639`).

Every path decides "is it still in use" from position, session or request history. None asks the
source.

## 3. The proposal

### 3.1 What is registered, and by whom

**On the buffer: its users.** A user is a small object with a name. `AudioBuffer.users` lists
them.

* The buffer registers two kinds by itself:
  * whoever waits in `get_buffer` for its first audio (`audio_buffer.py:517-520`, `:540-541`);
  * every open read stream: `get_raw_stream` registers on entry and removes itself in a
    `finally` (`audio_buffer.py:244-275`). A stream that is open and not reading (a prefetch that
    has its target, the parked rest of a fade) is still a user.
* `get_queue_item_stream` holds one user for its whole life, attached at the moment the buffer is
  taken (passed into `get_audio_buffer`, so it also moves along when a source at its limit is
  replaced by another one, `audio.py:4117-4156`). This closes the gap between "has the buffer"
  and "first read" for all five kinds of reader at once.
* One consumer registers itself: the flow generator holds a user of its own on the track it
  selected, from `load_next_queue_item` returning (`audio.py:2672`) until it selects the next one
  or ends (released next to `incoming_prefetcher.close()`, `audio.py:3277`).
* Analysis and the visualizer tap stay what they are: readers that never keep a source.

**On the provider: the sources that hold its slots.** `acquire_stream_slot` takes the holder,
adds it after the acquire and removes it in the same `finally` that returns the slot
(`models/music_provider.py:322-345`). It is exact where R13 infers.

**On the queue: nothing.** The queue answers one question from what it already keeps:
`plays_now_or_next(item)` is true when the queue has a session and the item is its current item,
the item its player holds (`index_in_buffer`), or the item that follows either of those according
to `get_next_item` (so repeat, unavailable items and a sleep timer are honoured).

### 3.2 In use, needed, and the rule

* **In use**: the source has at least one user.
* **Needed**: in use, or it is still the buffer of its item and `plays_now_or_next(item)` is true.
* **The rule**: a source that still fills and is not needed is stopped
  * always, when its item is on no queue, its queue has no session, or another buffer took its
    place on the item;
  * otherwise (its item waits further down the queue) only when its provider has no free slot.

"Stopped" is `AudioBuffer.clear()`. The buffer stays attached and cancelled, as
`_abort_source_buffer` leaves it today (`queue_loader.py:1609-1611`), so the next use replaces it.

The second half keeps what `dev` does for the successor at a start (R11) and what pull request
7762 intended: a prepared track that only moved down keeps its audio while nobody needs its slot.

### 3.3 When the rule is checked

One function, `AudioBuffer.release_if_unneeded()`, called from three places:

1. **A track asks for a slot and the provider has none free.** In `_get_audio_buffer`, before the
   refusal and the wait (`audio.py:4085-4091`): every holder of that provider's slots is asked,
   and the stops are awaited. Holders of other queues are asked too.
2. **The last user of a source leaves.** Inside the buffer.
3. **The items of a queue change.** In `update_items`, for the buffers of the previous items.

Moment 1 is the one that hands a slot over. It needs no hook in any queue edit, so it also
covers the plugins, Smart Fades ordering and whatever edits are added later. Moment 2 serves the
track that is already waiting when the stale reader ends. Moment 3 makes the release prompt.

### 3.4 How a slot gets to the track that needs it

There is no handover object. The stopped producer returns its slot to the provider's semaphore
(`models/music_provider.py:345`); the track that needs it either waits in `acquire_stream_slot` already
and is woken, or acquires right after moment 1 awaited the stop. The limit is never passed: the
design only ever stops sources of our own, and every open still goes through the semaphore.

### 3.5 Every way a user can end

| End | What happens |
|---|---|
| The stream plays to its end | the read stream's `finally` removes the user |
| The speaker disconnects | the route closes the encoder (`streams/controller.py:1267`), whose feeder closes its input (`helpers/ffmpeg.py:321-322`), which runs the `finally` |
| The source fails | the read raises, the generator ends through its `finally` |
| The task is cancelled | the cancellation lands in the generator, same `finally` |
| The stream is superseded | the route breaks out (`streams/controller.py:1269-1277`), the flow generator returns (`audio.py:2931-2936`); both close the stream |
| A wait for the buffer ends, with audio, a timeout or an error | the wait removes itself |
| A flow generator ends, in whatever way | its outer `finally` releases its claim (`audio.py:3276-3277`) |
| A transfer | a user is attached to one buffer object. When it ends late it can only touch that object, which the source queue's stop already cleared. The target's buffer has its own users |
| A user that never ends (a leaked generator) | Python closes an abandoned generator when it is collected (assumption, not probed). Until then the source is kept. The forced releases below and the inactivity timeout do not ask for users |

When the last user leaves, moment 2 runs.

**Forced releases stay as they are** and do not ask for users, because they end the playback the
users belong to: stop, clear, replace, end of queue, detach and remove (R8, R9), the start of an
item (R11), the paused handover (R13), a failed buffer (R3) and the inactivity timeout (R4).

### 3.6 What it replaces, what stays, what can go

**Not needed any more (all of pull request 7762's additions):** `flow_incoming_item_id`,
`release_left_item_source` with its buffer identity check and its "on no queue" check, the
release at the end of a stream request, `drop_fade_into`, and `_release_superseded_sources`
reading `requested_item_ids` and `fade_into_pending`.

**Gaps on `dev` that close without a special case:**

* R12 cannot find an item that left the queue. The rule reaches a source through its users and
  through the provider's holders, never through a queue lookup.
* The request path looks for a holder only among the items the speaker asked for (the second
  observation in the appendix). Moment 1 asks the sources that actually hold the slots.
* R5 and R6 stop working when the preparation task ends. The wait of the preparation is a user;
  what happens to the source afterwards no longer depends on a task still running.

**Can be deleted (step 3):**

* R5, the check after the preparation: the preparation's wait is a user, and when it ends a
  source whose item left the queue or whose queue has no session is not needed.
* R2's "last access more than 30 s ago" guess and its wait for the inactivity timeout: a replaced
  buffer with no user is stopped at once, one with a user when that user ends.
* R10's outright clear of the discarded tail: the rule in `update_items` covers it and spares a
  track a flow stream holds.
* R13's inference of who holds a slot: read the provider's holders instead.

**Stays:**

* R12 and its request history. It answers another question: "did the player move on by itself
  from the track the queue still shows as current?" Position cannot tell, only the requests can.
* R11 at the start of an item: a start beats everything else on its queue, including its own
  prepared successor.
* The session stamp, `index_in_buffer`, `next_item_id_preparing` (it keeps one preparation per
  queue), `next_item_id_enqueued`, and the fade markers `_crossfade_pending` and
  `_crossfade_handover`. They keep their own jobs; none of them decides a release in the new code.

### 3.7 Two small extras that do not follow from the rule

* **With step 2: a stream opens no new source for an item that is on no queue.** One check in
  `_get_audio_buffer`, next to the one for a consumer that left (`audio.py:4052-4057`). Without
  it a speaker that stays connected to a track removed at the edit would have the request open a
  fresh source for audio nobody will hear. It matches what the route already answers when the
  request comes a moment later (`streams/controller.py:935-937`).
* **Optional, with step 2: the preparation checks `plays_now_or_next` before it opens the
  source.** Without it the rule still stops the source (probe), after one wasted open.

## 4. Proof against the known sequences

"Rule" means it follows from 3.2 and 3.3 with the registrations of 3.1. "Extra" names what else
is needed.

| Sequence | How the design handles it | Follows from |
|---|---|---|
| **Incident: replace next**, one slot, playback pace | the prepared track has no user and is on no queue. Step 1: stopped when the new next track asks for the slot. Step 2: stopped at the edit | Rule. Probed |
| **Incident: play next** | the prepared track is still on the queue, no longer next, its provider is full: stopped as above | Rule. Probed |
| **Incident: remove** | as replace next | Rule. Probed |
| **Incident: move** | as play next | Rule. Probed |
| The request path aborts the abandoned request and not the capture | moment 1 asks the provider's real holders | Rule |
| **1.** Flow mode, "play next" while the fade into the incoming track is built | the flow generator's claim makes the track in use | Rule, given the one explicit claim of 3.1. The round 1 test passes |
| **2.** Repeat: the prepared track across the wrap | "next" is `get_next_item`, no index comparison | Rule. Probed |
| **3.** A removed track with an open stream request | the reader is a user. When the request ends, in whatever way, moment 2 stops the source; a track already waiting gets the slot | Rule. Probed, including the waiting track |
| **4.** A superseded flow stream overwrote the marker | there is no shared marker; a generator can only release its own claim | Rule. The round 2 test passes |
| **5.** Transfer: the cleanup of an old request cleared the target's buffer | a user's end can only touch the buffer it was attached to | Rule. Probed, with the transfer built by hand. The round 3 identity case holds for the same reason |
| **6.** A second flow generator in the same session | as 4: claims are per generator, not per session | Rule. Probed |
| **7.** A request that produced no output leaves a removed, prepared source | it never registered a user: the source is not needed from the edit on. If it had taken the buffer, it is a user until the request ends | Rule. Probed both ways. Extra 3.7 (first) for a speaker that stays connected |
| **8.** A flow stream ends while it protects a removed incoming source | the claim goes in the generator's `finally`, moment 2 stops the source | Rule. Probed |
| Window: a moved track that a fade already leads into | outside flow mode the track a fade is mixed into is `index_in_buffer` from its selection (`audio.py:2225`, before the pending marker at `:2265`), so it cannot be moved or removed (`controller.py:864-867`, `:937-942`). In flow mode the generator's claim holds it. A declared overlap claims the hand-off at the start of its item (`audio.py:2121-2124`) and reads nothing until the end | Rule. Nothing fade-specific is needed. From code reading; the test of pull request 7762 for this sets the marker by hand |
| Window: "next" lands while the preparation still fetches details | the preparation opens the source, its wait is a user, and when it ends the source is not needed: stopped | Rule. Probed. Extra 3.7 (second) avoids the wasted open |
| Window: the sweep at the next request does nothing for a track that left the queue | see 3 | Rule |
| Window: a dynamic rebuild in flow mode drops the track the flow fades into | with step 3 the rebuild no longer clears a source a flow stream holds, so the track plays in full. **Not solved:** after that track the flow asks for "next after it", the track is no longer on the queue, and the flow ends as if the queue ran out (`controller.py:1784-1787`), then resumes when the player goes idle | Half. The source: rule. The continuation: unchanged from `dev` |
| Window: the "Aborting the source of ..." line also fires for a source without a slot | the provider knows its holders, so the line can be limited to them | The holder set, step 3. Not the rule |
| Window: a request waits 15 s for a slot while a speaker may give up after 10 | not addressed. With the rule a request only waits for a source that is needed, so the wait is right; the number is open question 3 | Not covered |

## 5. Alternatives

**(a) The narrow variant.** In `update_items`: when the item in `next_item_id_preparing` left the
queue or no longer plays next, the queue is not in flow mode and no request is open for it, abort
its source.

* Cost: about 30 lines and a handful of tests. No risk in flow mode, because it does nothing
  there.
* It ends the incident for a per-item player that has not fetched the next track yet.
* It leaves open: flow mode (the same preparation, the same stale capture), a player that
  fetched the next track early (its source is not released, also not when that request ends:
  finding 3),
  the window while the preparation fetches details, the search for the real holder at the next
  request. It still reads `requested_item_ids`, which does not see direct PCM consumers.
* Every later widening is another marker.

**(b) One more round on pull request 7762.** Fix findings 6, 7 and 8 on top of its last head.

* Finding 6 needs the marker per generator. Finding 7 needs a record of removed items that a
  request still holds. Finding 8 needs a release when a flow stream ends. Each is a piece of the
  user set, built as a special case for one consumer.
* Cost: an estimated 80 to 120 lines on top of its 141, a fifth review round, and the same kind
  of gap for the next consumer nobody listed (the prefetcher and the direct PCM streams have no
  marker today).
* It leaves all six windows as they are.

**(c) The proposal.** Users on the buffer, holders on the provider, one rule.

* Cost: about 150 lines in step 1, three pull requests in total, some 60 existing tests whose
  stand-ins need a new member.
* It leaves open: the half of the dynamic rebuild window, the 15 s wait, and the open
  questions below.

**Chosen: (c).** Four rounds showed that the hard part is "is it in use", not "release
it". (a) and (b) keep answering that from markers per kind of user. (c) answers it in the two
places every user passes through, so a new kind of consumer is covered without knowing about it.
Its first step has a blast radius close to (a): it stops nothing that is reachable unless a
provider is out of slots.

## 6. Steps

These take the place of pull request 7762, which is closed. Its test rig and its incident
tests are reused; its four tests that set a marker by hand are rewritten with a real reader.

### Step 1: a track that needs a stream slot gets it from a source nobody needs

* Contents: users on the buffer (3.1), the user of `get_queue_item_stream`, the flow generator's
  claim, the holders on the provider, `plays_now_or_next`, and the rule at moments 1 and 2. No
  change in any queue edit path.
* Size: about 150 source lines in `audio_buffer.py`, `audio.py`, `models/music_provider.py` and the
  queue controller; about 350 test lines; one-line changes to the stand-in helpers of five test
  files.
* **Ends the incident: yes, in all four forms.** Probed: with the edit hook switched off every
  incident test passes. The stale capture lives on until the next track asks for the slot, then
  gives way within the stop.
* Safe on its own: it stops a source in two cases only. Its provider has no free slot and the
  source is not needed; or its last user leaves and nothing can reach it any more (its track is
  on no queue, or a seek replaced its buffer). A replaced buffer is never read again
  (`audio.py:1778-1782`). On a provider with a slot to spare nothing else changes.
* Unit tests: the incident tests; a reader, a waiting request, a prefetch and a flow claim each
  keep a source that left the queue, and its end releases it; a boundary mix that reads the next
  track registers as its user; two
  flow generators in one session; a transfer; a prepared track that still plays next is never
  taken, by a request of its own queue or of another one; a source that is in use is never taken;
  one test per kind of consumer that asserts `users` while it reads.
* Lab, on an account with one slot that delivers at playback pace (assumes the lab has one):
  * a per-item player (Squeezelite, DLNA): in the last 30 seconds of a track do replace next,
    play next, remove next, move next, five edits in a row. The next track starts and the log has
    no stream limit error.
  * the same on a player set to flow mode with crossfade, with the edit inside the last 15
    seconds. The incoming track plays in full.
  * repeat all, play next on the last track.
  * a player that fetches the next track early: the edit after that fetch.
  * two players on one account: a stale source on the first, a start on the second (it starts);
    the first really playing, a start on the second (refused, as today).
  * a transfer to another player while playing, then an edit.

### Step 2: a track taken off the queue stops loading at once

* Contents: moment 3 in `update_items`, the first extra of 3.7, and optionally the second.
* Size: about 40 source lines, about 150 test lines.
* Not needed for the incident. It stops a playback-pace source from playing a track to the
  service that nobody hears, and frees the audio of removed items at the edit instead of at the
  next request. From code reading, not probed: it also releases what a transfer leaves behind on
  a target that was playing (`transfer_queue` loads the new items at `controller.py:1531` without
  releasing the target's old audio).
* Unit tests: the four edits release at the edit; an edit that keeps the next track releases
  nothing; a removed track with a connected speaker opens no new source; every caller of
  `update_items` listed in section 2 with a prepared next track.
* Lab: the step 1 runs again, plus a count of source opens per track on a provider with several
  slots while a plugin splices clips in front of the prepared track. One open per track, as
  before.

### Step 3: delete what the rule covers

* Contents: R5, the 30 second guess in R2, the outright clear in R10, `_held_stream_slots`
  reading the provider's holders, the log line limited to real holders, the two README sections.
* Size: about 40 lines out, 30 in.
* Lab: the regression runs (seek, skip, pause past the pause timeout, a paused queue handing its
  slot over, a dynamic queue with an enqueue in flow mode).

## 7. Risks and open questions

**Risks.**

* A reader that bypasses `get_raw_stream` would be invisible. There is none for playback today;
  the test per kind of consumer pins that.
* A wrong "no" from `plays_now_or_next` costs a cold start of that track on a full provider. A
  wrong "yes" is today's behaviour. Only a missing user can cut audio, and that is now one
  mechanism in one file.
* A user that never ends keeps a source that is not needed. The forced releases and the
  inactivity timeout remain as the backstop. `users` belongs in the streams diagnostics and in
  the debug line when a wait for a slot starts, so a lab run shows who holds what.
* The prototype proved the rule, not the code. The real change has not been written or reviewed.

**Open questions, as choices** (the maintainer's answers are under Decisions).

1. **A prepared track that only moved down the queue.** (a) It gives way only when its provider
   has no free slot: the proposal, as `dev` does at a start. (b) A playback-pace source always
   gives way, because the service is told the track plays. Recommended: (a), and (b) only
   if a service shows the wrong track as playing.
2. **A speaker asks for a track that is not the next one while the next one is prepared** (it
   jumped by itself). The rule keeps the prepared track, since it plays next by position, so the
   request waits as on `dev`. (a) Leave it until someone reports it. (b) Within one queue, a
   track a speaker asked for beats a prepared track nobody asked for. Recommended: (a).
3. **15 seconds against a speaker's 10.** (a) Leave `STREAM_SLOT_PLAYBACK_WAIT_TIMEOUT`. (b)
   Lower it below the shortest speaker timeout, as its own small change after the lab measured
   which players give up when. Recommended: (b), after step 1.
4. **Step 1 and step 2 together or apart.** Apart means a night in which a removed track keeps
   capturing until the next track asks for the slot. Recommended: apart, step 1 first, step 2
   after its lab run.
5. **The new next track starts cold after an edit**, as after a skip: nothing prepares it again
   when the edit comes after the preparation trigger. (a) Accept. (b) Start the preparation again
   from the edit. Recommended: (a), and (b) only if the lab hears a gap at a gapless boundary.
6. **Seen while testing, not caused or fixed by this:** in flow mode a "play next" that lands
   after the flow selected its next track and before that track's first chunk is passed over by
   that flow: it plays the selected track and continues after it. The round 1 test shows it (the
   flow emits both tracks and never the inserted one); not checked on a running server. (a)
   Leave it. (b) A follow-up that makes the flow commit its selection in the queue's position, as
   the item route does (`audio.py:2225`). Recommended: a probe in the lab first.

## Appendix: what four review rounds found in pull request 7762

### The defect

A music provider can limit how many source streams it serves at once
(`max_concurrent_streams`); some allow one. A source that is being read holds a slot, and a
provider that delivers audio at playback pace holds it for the length of the track. The
queue prepares the next track ahead of time.

A queue edit that keeps the current track playing ("replace next", "play next", and from
code reading removing or moving the upcoming track) goes through
`PlayerQueuesController.update_items`, which swaps the item list and releases nothing. The
capture of the prepared track that left the queue, or is no longer next, keeps the slot
until the track has run out. Every track the speaker asks for meanwhile waits for the slot
and is dropped. On a one-slot provider at playback pace that lasted eight tracks in the
incident this came from. A second observation: the request path only looks for a holder
among the items the speaker itself asked for, so it aborts the abandoned request (which
never held the slot) and not the capture nobody needs.

The tests that prove it: `tests/controllers/player_queues/test_stale_capture_slot.py` on the
branch (the cases named `test_the_track_that_plays_next_after_a_queue_change_gets_the_slot`
fail on `dev` with `ProviderStreamLimitError`).

### The approach of pull request 7762

`update_items` keeps the previous item list and calls `_release_superseded_sources`, which
releases the still-filling sources of items that left the queue and, when its provider has
no free slot, that of the prepared item that no longer plays next. Whether a source is
still in use is inferred from separate markers: open stream requests
(`requested_item_ids`), queue positions (`current_index`, `index_in_buffer`), a pending
crossfade (`fade_into_pending`, `drop_fade_into`), and, added along the way, a field
`PlayerQueueData.flow_incoming_item_id` that the flow generator writes, a release at the end
of a stream request (`release_left_item_source`), a buffer identity check and an
"on no queue at all" check.

### Findings per round (each reproduced by the reviewer)

Round 1, on the first version:

1. Flow mode, a regression: while a flow stream has selected its incoming track and is
   still building the crossfade into it, `index_in_buffer` names the track before it, so
   "play next" released the incoming track's source and the track was skipped or cut short.
2. Repeat: the prepared track across the wrap failed an index comparison and kept the slot.
3. A removed track with an open stream request was never released when that request ended
   (this one exists on `dev` too).

Round 2, after the fixes for round 1:

4. A flow stream that was superseded by a newer session wrote `flow_incoming_item_id` after
   its load returned and so overwrote the marker of the active stream; "play next" then
   cleared the source the active stream was fading into.
5. `transfer_queue` hands the same `QueueItem` objects to the target queue; the cleanup of
   an old stream request of the source queue then cleared the target's new buffer.

Round 3, after the fixes for round 2:

6. Blocker. The marker guard tells sessions apart, not producers within one session. A
   reconnect can start a second flow generator in the same session while the first still
   awaits its next track; when that load returns, the old generator overwrites the active
   one's marker (`music_assistant/controllers/streams/audio.py`, the assignment of
   `flow_incoming_item_id`). A following "play next" clears the source the active flow is
   building its crossfade into. A probe with the real generator delivered 32 seconds
   instead of 62.
7. A request that produced no output leaves a removed, prepared source behind: the
   preparation finished, the speaker requested the track, the request was suspended while
   resolving source access, "replace next" removed the track (its source kept because of
   the open request), the speaker disconnected, and the request returned without audio. The
   cleanup has no buffer to release, and nothing else can find the removed item. The slot
   stays taken until the source ends or the inactivity cleanup of about five minutes.
8. Ending a flow stream does not release a removed incoming source it protected: the flow
   selects B and builds its fade, "replace next" removes B (kept for that flow), the flow
   disconnects during the build. B's capture keeps running and its marker stays published;
   no queue cleanup can reach B any more.

The reviewer of round 3 also showed a case in which the buffer identity check alone
decides (the target of a transfer removes a still-open prefetch before the old source's
cleanup runs), which the author had thought impossible.

### Windows left open on purpose

- A moved track that a fade already leads into keeps its capture.
- A "next" that lands while the preparation still fetches stream details.
- The sweep at the speaker's next request does nothing for a track that left the queue.
- A dynamic queue's tail rebuild in flow mode can drop the track the flow stream is fading
  into, as it does on `dev`.
- The "Aborting the source of ..." log line also fires for a source that never got a slot.
- A stream request waits 15 seconds for a slot while a speaker may give up after 10.

### The pattern

Every fix added shared state or a release path whose own life cycle then had a gap. Nothing
in the code answers the question "who is using this source right now" directly; it is
inferred, in several places, from markers that each cover one kind of user.
