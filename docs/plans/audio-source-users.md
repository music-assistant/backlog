# A slot in the name of a queue, and the audio in a queue's scope: technical plan

Status: decided on 2026-11-17 and being built as five pull requests (see Steps). Owner: Marcel
van der Veldt. This document is the technical companion of the story "Playback stalls after a
queue change on a music service that allows one stream at a time" (#481) on the project board.

It follows a first plan of 2026-11-16, "an audio source knows who uses it", whose first step
landed as music-assistant/server#7800: an audio buffer knows who reads it, and a track that
needs a service's only stream gets it from a prepared track nobody needs. This plan replaces
the two steps that were to follow it. Two reviews judged earlier versions of it not sound;
the slot half was then simplified, and a third review judged this version sound with changes,
which are worked in. Section 9 lists what changed.

Written against server `dev` at `fd2712c98`; every `path:line` is on that commit.

Paths are relative to `music_assistant/`. The folder is left out for `audio.py`,
`audio_buffer.py` (in `controllers/streams/`) and for `stream_feeder.py`, `queue_loader.py`,
`playback_tracker.py` (in `controllers/player_queues/`). A bare `controller.py` is the queue
controller; the streams controller is written `streams/controller.py`.

## Summary

**The slot.** A slot stays what it is today: one open source under the provider's limit, with
today's numbers. Two things are added. The holder is a fact: the buffer that holds a slot
knows the queue and the item it was made for. And when a source of a queue gives its slot
back while that queue still plays, the slot stays reserved for that queue for a few seconds;
to every other taker it is busy that much longer. No queue is admitted or counted, and no new
kind of refusal is added.

**The audio.** A queue has a record of the buffers made for its items. A buffer is in scope
while a stream reads it, or while it is the buffer of the track that plays, the next one, a
claimed one, or a fully loaded previous one. One function drops what is out of scope at once,
on every provider, filling or complete. A queue without a session has no scope.

**Does it hold up?** The cleanup half came through two reviews with small findings, all
answered below. The slot half is new in this form and small; the race it is for is measured
(P8, P10). No code of the design exists.

**Accepted on purpose.** On a source that loads ahead the slot is free once the track has
loaded, so a second speaker can play from a one-stream service at the same time, as today
(P1a).

**Steps.** Four pull requests and one small one, about 2,000 lines of diff with their tests.

## What was checked

A virtualenv was built in the checkout. Two rounds of throwaway probes ran with the rig of
`tests/controllers/streams/stream_slot_rig.py` (`pytest <file> -p no:randomly --no-cov`; all
passed; the files are removed): twelve on `2211a9a39`, five on `fd2712c98`. What `dev` does:

| Probe | Sequence | Measured on `dev` |
|---|---|---|
| P1a | two queues, one-stream provider, sources that load ahead | the second is not refused; both streams deliver side by side |
| P2 | a reader on a buffer, a second request for a position the buffer cannot serve | one slot: the first buffer is cancelled and its reader ends. Two slots, no limit: the buffer stays and the reader reads on |
| P3 | the preparation waits in the format probe, Replace Next removes the track | the buffer is made after the removal and cleaned by R5 when it is ready. Without R5 and without a limit it keeps filling |
| P6 | the next track is prepared, repeat one is switched on | the prepared capture keeps filling and keeps the only slot |
| P7 | queue B asks while queue A's realtime source holds the only slot | refused at once: "already playing on Queue A" |
| **P8** | **A's realtime source ends; before A's next source opens, B asks** | **B gets the slot. A's preparation fails ("already playing on Queue B"), no source is opened for A's next track, and its speaker's request for it is refused the same way** |
| P9 | a preview already waits for the slot when A's realtime source ends | the preview gets the slot; A's next track has a buffer and no audio until the preview ends |
| P10 | A skips: its playing source is stopped; before the new one asks, B asks | B gets the slot; A's start is refused, which on a real start stops A (`controller.py:2668-2685`) |
| P11 | A's source loads ahead and holds the slot; the preparation of the next track waits for it; Replace Next removes that track; A's source ends | the removal leaves the waiting buffer alone (its user is the preparation's wait); its source then starts, for a track that is gone; R5 cleans it afterwards |

P8 is the race the reservation is for: a queue that moves from one track to the next loses its
service to a queue that asks in that moment. P7 shows it is the only way in for another
queue: a moment earlier or later it is refused.

How long that moment lasts could not be measured without a server: in the rig it is a few
turns of the event loop. From the code it is the time from the release
(`audio_buffer.py:434-455`) through `track_fully_buffered` (`stream_feeder.py:179-198`), the
preparation task, `load_next_queue_item` with its library lookups and stream details
(`controller.py:1779`), to the new producer asking (`audio.py:908-914`). Stream details are
normally in hand from the preload; when they have to be fetched it is a provider round trip.
At a skip it is the same from R11's stop (`queue_loader.py:456-460`) to the start's request
(`:496-501`).

## 1. The slot

### 1.1 What stays

A slot is one open source. `max_concurrent_streams` keeps its values and its meaning, the
number of source streams (`models/music_provider.py:307-318`). It is taken in
`get_media_stream` for as long as the source is open (`audio.py:908-916`). The waits keep
their budgets: 5 s for a preparation, 15 s for playback (`streams/constants.py:221`, `:228`).
The refusal of a start behind a realtime source (`stream_feeder.py:209-236`), the hand-over
by a paused queue (R13, `:268-301`), a start on a full provider (R11) and a speaker that
moved on by itself (R12) behave as today.

### 1.2 The holder is a fact

Today the provider lists the buffers that hold its slots (`stream_slot_holders`,
`models/music_provider.py:325-328`), and a buffer does not know whose it is. Step 1 lets a
buffer carry the queue id and the queue item it was made for, set when `get_buffer` reports
it to the queue (2.1). Then "who holds a slot" is read, not inferred:

```python
for buffer in provider.stream_slot_holders:   # holds a slot right now
    buffer.queue_id, buffer.queue_item        # for this queue and this track
```

The queue's own record of its buffers (2.1) is not enough for this: it would have to be
searched in every queue.

Takers without a buffer: the format probe takes its seat in the name of its item's queue
(`audio_buffer.py:1143-1149`); a preview (`streams/controller.py:2095`) and the Ynison plugin
(`providers/yandex_ynison/provider.py:866`) have no queue.

### 1.3 The reservation

`acquire_stream_slot` gets the queue it is taken for (`models/music_provider.py:330-361`).

* **Made** when a source of queue Q that delivers at playback pace (`AudioBuffer.is_realtime`,
  `audio_buffer.py:157`) gives its slot back, Q still has a session, and Q has something to
  go on with on that provider: one of its playing, next or claimed items, other than the item
  whose source just delivered everything, has stream details from that provider or none yet.
  One per queue and provider. So none after the last track of a queue, none when the next
  track is from elsewhere, and none on a source that loads ahead.
* **Lasts** `STREAM_SLOT_RESERVATION` seconds, then lapses by itself.
* **Used** by Q's next request for a slot of that provider: it takes the slot at once and the
  reservation is gone.
* **For everyone else** the slot counts as taken: a taker that is not Q waits while the open
  sources plus the reservations of other queues reach the limit.
* **Ends early** when Q's session ends (`_end_session`, `controller.py:2655-2666`, only when
  it really cleared the session): a stop, the pause timeout (`:1124`), a failed start, a
  paused queue that hands over, the stop of a transfer (`:1500`). A release that comes after
  the session ended makes none. A player that is detached or removed: nothing to do, it lapses.

**How it is built.** The semaphore cannot do this: a taker that is already in line gets a slot
the moment it is returned (P9). So the limit becomes a counter with a condition in the same
place: same number, same waits, same errors, and the one extra clause above in what a waiter
waits for. Nothing outside `MusicProvider` touches the semaphore object today.
`has_available_stream_slot` keeps its meaning, "fewer sources are open than the limit", so
everything a queue decides for itself from it (R2, R11, the skip of a preparation at
`stream_feeder.py:558-583`) is as today.

**What another taker sees.** No new kind of error. Its request waits inside its own budget and
gets the slot when the reservation lapses or after Q's next source ended. If Q's next source
takes the slot meanwhile, the request waits its budget out and ends with today's limit error.

**A taker that already waits when the source ends**, a preview for one. Today it gets the
slot that moment, ahead of the queue's next track (P9). With a reservation the queue's next
track goes first. A preview has 5 s in total (`streams/controller.py:2095`,
`streams/constants.py:221`), so it then ends in the limit error where it slipped in today.
That is a change, and the point of the reservation. Where no reservation is made (the last
track, a next track from elsewhere) the waiter gets the slot as today. A reservation that is
made and then not used, because the next track failed to load, costs a waiter up to its
length, which can be all the budget it had left.

**How long.** Decided: it starts at 5 s and is corrected from what the lab measures. It has to
be longer than the moment of P8 and shorter than another queue's playback budget of 15 s.

### 1.4 Inside one queue the track it needs wins

The queue never waits for itself:

* its own reservation is free to it;
* before it opens a source, the queue's function drops what is out of its scope and nobody
  reads (2.4);
* at a skip or a seek the paths of today stop the source that is in the way, read or not:
  R11 at a start (`queue_loader.py:1534-1570`), R2 for a replaced buffer on a full provider
  (`audio_buffer.py:579-586`), R12 for a speaker that moved on.

What is new at a skip is only that the slot R11 frees cannot be taken by another queue before
the new track asks for it (P10).

### 1.5 What the fact simplifies

Only one thing is a plain replacement. `_held_stream_slots` (`stream_feeder.py:614-640`)
walks the items of every queue for a buffer that "fills and is ready" to guess who holds a
slot. It becomes: the provider's holders whose audio is ready (today's condition, `:638`),
each with its queue id, for the queues other than the asking one that have a session. Its
three users keep their logic:

| User | Change |
|---|---|
| `refuse_held_stream_slot` (`:209-236`) | reads `buffer.is_realtime` instead of the item's details |
| `_paused_stream_slot_holder` (`:585-601`) | none; it only uses the queue id |
| `release_paused_stream_slot` (`:268-301`) | none |

One case comes out differently: a holder whose item left the queue's list is found. A holder
that has its slot and no audio yet is not refused behind, as today: without the readiness
condition a start would be refused at once where today it waits and gets the slot when that
source fails. A buffer that still waits for a slot is not a holder in either version.

## 2. The audio in a queue's scope

### 2.1 What the queue knows

* **Its buffers.** `PlayerQueueData` gets a runtime-only record `(queue item, buffer)`.
  `get_buffer` reports every new buffer to the queue, which also puts the queue id and the
  item on the buffer (1.2). An entry whose buffer was cleared is dropped when the next buffer
  is recorded and whenever the record is walked, so it never grows beyond the buffers that
  still hold audio. Today a buffer whose item left the list is out of reach
  (`stream_feeder.py:140-145`, `queue_loader.py:1441`).
* **Its position**: `current_index` and `index_in_buffer`.
* **The claims of its streams**, as on `dev` (`audio.py:425`, `:4358`): the item a stream
  decided to play, for the session it was made in.
* **The readers of a buffer**, as on `dev` (`audio_buffer.py:94-127`): the `BufferUser` a
  stream holds from the moment it takes the buffer, and every open read.

### 2.2 Definition

An item of a queue with a session is **playing** (at `current_index` or `index_in_buffer`),
**next** (`get_next_item` of either; with playing this is `plays_now_or_next`,
`controller.py:2388-2416`), **claimed** (a stream of the current session decided to play it,
on the queue or not), or **previous** (before `index_in_buffer`, for audio that has fully
arrived only; the window R7 keeps, `stream_feeder.py:470-473`).

A buffer is **in scope** when

1. a stream reads it or waits to read it; or
2. it is still the buffer of its item, and that item is playing, next, claimed or previous.

A queue without a session has no scope: no audio is made for it (2.5). The branch of
`plays_now_or_next` for such a queue (`controller.py:2405-2407`) goes, with the lobby preload
of the music quiz, its only user (2.7).

**A wait is not a reader.** A caller that only waits for a buffer to become ready, the
preparation or a start, is a stand-in user today (`audio_buffer.py:622-625`, `:653`), and that
stand-in keeps a buffer whose track is gone (P11). It stops counting for rule 1. When such a
wait ends the check still runs, as today. A stream that waits for its own audio passes its
`BufferUser` in and counts.

**The track a start is trying is playing.** A start that fails on an item with an ordinary
audio error moves on to the next one in its own loop (`controller.py:3236-3317`), while
`index_in_buffer` still names the first (`:3174`) and `current_index` only moves after a load
(`:3249`). The start moves `index_in_buffer` to the item it tries before each load, so that
item is in scope through the retries and the check of 2.5 does not turn it away.

### 2.3 The one function

`PlayerQueuesController._enforce_scope(queue_data)` walks the record and asks every buffer to
let go if it is out of scope. For one buffer that is `release_if_unneeded`
(`audio_buffer.py:257-280`), with three changes: the queue's scope answers `may_release` (in
place of `_may_release_source` and its free-slot gate, `audio.py:4326-4356`, `:4337-4339`); a
buffer whose source is done is freed too (today it returns at `:270`); and the release wakes
whoever waits for the buffer, so a preparation that is no longer needed ends there instead of
at its timeout (`audio_buffer.py:660-702`). The decision and the mark that makes the buffer
invalid are one synchronous step (`:278`); it is detached from its item and cleared in a task.

It never touches a buffer a stream reads.

### 2.4 When it runs

| Moment | Where |
|---|---|
| A session starts; a session ends | `_play_index` after `controller.py:3203`; `_end_session` (`:2655`) |
| The items change | `update_items` (`controller.py:2118`): every edit, load, plugin splice, clear, restore, transfer, and shuffle (`:2989`) |
| The position changes | `track_loaded_in_buffer` (`controller.py:1855`), the fade selection (`audio.py:2351`), another current item (`playback_tracker.py:332`), `mark_ended` (`controller.py:963`) |
| What plays next changes otherwise | see below |
| A claim ends or moves | `_claim_item` (`audio.py:4358`), as on `dev` |
| The last reader leaves; a wait ends | `_drop_user` (`audio_buffer.py:635-638`), as on `dev` |
| A buffer is replaced | `get_buffer`, where it detaches the old one (`audio_buffer.py:574`): that buffer is asked at once, also when it was never read and the provider has a slot to spare. A reader keeps it; R2's release on a full provider stays |
| The queue is about to get audio | `_get_audio_buffer`, before the candidates are tried, and the checks of 2.5 |

**What plays next without the items or the position changing.** `get_next_item` reads four
things (`queue_loader.py:635-672`, `controller.py:2365`): the list, the repeat mode, the sleep
timer target and whether an item is available.

| What | Where it is written | Moment |
|---|---|---|
| Repeat | `set_repeat` (`controller.py:555`) | yes. The two other writers are followed by a load of items (a transfer, entering dynamic mode) |
| Sleep timer target set | `set_sleep_timer_target` (`controller.py:671`) | yes |
| Sleep timer target moved to the item that plays now | `_repin_sleep_timer` (`:2847`) | yes |
| Sleep timer target dropped | `_forget_sleep_timer_target` (`:2869-2878`), which the drop, the clear and the timer's own stop go through | yes |
| An item marked unavailable | `load_next_queue_item` (`controller.py:1836-1838`) | already ends in `update_items` |
| An item marked unavailable elsewhere | `audio.py:1822`, `:2075`, `streams/controller.py:1096`, `controller.py:3294` | no: what is left is that item's own failed buffer, which clears itself (R3) |
| Shuffle | reorders the items | `update_items` |

**What a missed moment costs.** Memory, until the next moment of that queue, at the latest
the end of its session or the inactivity timeout (R4). On a provider without a free slot it
also costs a slot until a track asks for one: the check of step 1 at that point stays
(`audio.py:4214`, `:4402-4431`) and asks with the queue's scope. P6 is that cost on `dev`.

### 2.5 The last check is at the source

The queue is asked the same question, "is there a session, and is this item playing, next or
claimed", at three places. Each sits after a wait and before the thing the wait was for.

| Where | After | Before | When the answer is no |
|---|---|---|---|
| `get_buffer` (`audio_buffer.py:605-607`) | the format probe returned | `_new_buffer` | no buffer; `_get_audio_buffer` returns None, as for a consumer that left (`audio.py:4217-4222`) |
| the format probe (`audio_buffer.py:1149`) | it has its seat | it opens the source to probe it | no probe, no buffer |
| the producer, in `get_media_stream` (`audio.py:914-916`) | it has its seat | the first read of the source, which is what opens it | the buffer lets go (2.3), nothing is opened, the seat goes back, and it does not count as a source that delivered everything (`audio_buffer.py:449-455`) |

The third is the one the second review asked for: `fill()` only schedules the producer
(`audio_buffer.py:478-480`), and the producer may wait for a seat long after the buffer was
made (P11). With 2.2 the buffer of P11 is released at the edit already, and its producer is
cancelled while it waits, without a seat; the check at the source covers an edit whose moment
was missed.

### 2.6 What forces a buffer out although it is read

The function never does. These paths do, and they stay as they are:

| Path | Why it may |
|---|---|
| A stop, a clear, a replace, the end of a queue (R8, `stream_feeder.py:509-556`); a detach, a removal (R9, `controller.py:2687`) | the playback its readers belong to is over; the server ends those streams itself (`streams/controller.py:1294-1302`, `audio.py:2779`) |
| A replaced buffer on a provider without a free slot (R2) | the replacement cannot start while the old source holds the slot (P2) |
| A start on a provider with a limit (R11); a speaker that moved on (R12); a paused queue that hands over (R13) | the track that plays needs the slot |
| A dynamic rebuild (R10, `queue_loader.py:1440-1445`) | today's behaviour, until step 5 |

### 2.7 The quiz lobby

Decided: the music quiz no longer buffers the first track of a round in its lobby.
`_warm_first_track` keeps its staging of the item (`providers/music_quiz/__init__.py:1549-1564`)
and loses the buffer call (`:1565-1575`) and the import only that call used (`:88`). Later
rounds are prepared inside a session (`:1622`) and are not touched. Nothing else makes audio
for a queue without a session.

## 3. What it replaces

| # | Today | In the design |
|---|---|---|
| R1 | the producer holds a slot and is listed | stays; the listed buffer knows its queue and item |
| R2 | a replaced buffer: cleared on a full provider, or after a 30 s guess that never fires (`audio_buffer.py:299-300`), else left for its reader | stays; the guess goes in step 4, the buffer goes when its reader leaves |
| R3, R4 | a failed buffer; 300 s without access | stay |
| R5 | after the preparation's wait: clear when the item left or the session ended (`stream_feeder.py:140-152`) | goes in step 4: 2.5, the edit, and the end of the preparation's wait cover it |
| R6 | a cancelled preparation clears its half-filled source (`stream_feeder.py:155-165`) | goes in step 4: its wait no longer keeps the buffer (2.2) |
| R7 | clears index <= current - 2 (`stream_feeder.py:466-507`) | goes in step 4: "the position changes" |
| R8, R9, R11, R12 | forced releases and arbitration | stay |
| R10 | a dynamic rebuild clears the discarded tail, also while read | goes in step 5, as its own change |
| R13 | a paused queue hands over; who holds is inferred | stays; the inference is replaced by the fact (1.5) |
| R14 | fade data | stays |

Of step 1 of the first design everything stays: the readers, the claims,
`release_if_unneeded`, the holder set, the check when a track needs a slot. What changes is
who answers `may_release` (the queue's scope, without the free-slot gate) and what counts as
a reader (2.2).

## 4. Proof against the known sequences

| Sequence | What happens | Follows from |
|---|---|---|
| **Incident**: replace next, play next, remove, move, on a one-stream provider at playback pace | the edit runs the function: prepared B is not next, nobody reads it, no stream claims it. Invalid, detached, stopped. Its slot is reserved for the queue, so the new next track gets it | Scope; reservation |
| Findings 1 to 8 of pull request 7762 | as in the earlier versions: the flow stream's claim (1, 4, 6, 8), `get_next_item` (2), the reader and the claim of an open request (3, 7), a queue walks only its own record (5) | Scope |
| A "next" that lands while the preparation fetches details, probes the format, or waits for a slot | 2.5: nothing is opened for it | Scope |
| A dynamic rebuild drops the track the flow fades into | as `dev` until step 5, then its audio is spared. **Not solved:** the flow then finds no successor and ends as if the queue ran out | Half |
| 7800: a buffer being released can be taken; a claim before the buffer exists; an old playback's claim after a transfer | invalid in the same step as the decision; the scope is asked per item; claims count for the current session | Scope |
| Review 1, 2: a buffer is replaced while a stream reads it | in scope by its reader; on a full provider R2 cuts it, as today (P2) | Scope; existing |
| Review 1, 3 and review 2, 2: audio made, or a source opened, after an edit | 2.5 and 2.2 (P3, P11) | Scope |
| Review 1, 6 and review 2, 4: repeat, the sleep timer | moments of 2.4 (P6) | Scope |
| **A queue moves to its next track while another asks** (P8) | the slot of the source that ended is reserved for the first queue; the other waits. The first queue's next source takes it | Reservation |
| **A queue skips while another asks** (P10) | the same, for the slot R11 freed | Reservation |
| A preview waits when the source ends (P9) | it keeps waiting; the queue's next source goes first | Reservation |
| Two queues on a one-stream provider at playback pace | the second is refused at once, as today (P7) | Existing |
| Two queues on a one-stream provider that loads ahead | both play, as today (P1a) | Accepted |
| Pause and resume; the pause timeout | nothing changes; the stop of the timeout ends the session, and with it scope and reservation | Existing |
| A paused queue whose slot another queue needs | stopped (R13); its session is over, so its release makes no reservation and the other queue gets the slot at once | Existing |
| A seek the buffer cannot serve, one stream | R2 clears the replaced source; its slot is reserved for the same queue | Existing; reservation |
| Crossfade and flow mode | both tracks in scope; sources one after the other on a one-stream provider, as today | Scope |
| A player that fetched the next track early; one that moves on by itself | its request reads and claims the track; R12 | Scope; existing |
| A transfer; a group whose leader changes | the source queue is stopped: no scope, no reservation. The target starts as any queue | Scope |
| A restore; a plugin that splices items in | `update_items` | Scope |
| A mixed queue that alternates providers | each track asks for a slot when it needs one, as today. A reservation lapses unused when the next track is from elsewhere | Existing |
| The quiz lobby | no preload; the first round starts as any play does | Decided |

**A request that was connected when its track was removed (decision 7).** It reads and has
claimed the track, so the track is in scope until that request ends, as on `dev`. A source for
an item that is not playing, next or claimed is not opened (2.5). The extra of the first
design is left out.

## 5. What users will notice

* **A speaker that plays from a service at playback pace keeps it across a track change.**
  Another speaker that asks in that moment no longer takes the stream away, which today costs
  the first speaker its next track or stops it at a skip (P8, P10).
* **For that other speaker the service is busy a few seconds longer.** It starts a little
  later, or gets the refusal it gets today. No new kind of refusal and no new text.
* **A preview that waits for such a service no longer slips in between two tracks.** It ends
  in the error it gets today at any other moment of the track. After the last track of a
  queue it gets the stream as today.
* **A prepared track that was moved down or replaced no longer blocks the track that plays
  next**, on every service. It is loaded again when its turn comes.
* **Memory**: the audio of a removed or passed track is freed at once when nobody reads it.
* **Music quiz**: the first round starts as any play does, without audio prepared in the lobby.

Unchanged: the limits and what they count, two speakers on a service that loads ahead,
crossfade and flow mode, pause and resume, a stream that still reads its audio.

## 6. Stored data and interfaces

* **Config entries, database rows, API commands, events, texts a user reads:** none changes.
  No migration.
* **`MusicProvider`:** `acquire_stream_slot` gets an optional queue id; the semaphore becomes a
  counter with a condition. `max_concurrent_streams` keeps its name, values and docstring. The
  Ynison protocol (`providers/yandex_ynison/protocols.py:49`) passes no queue and is not touched.
* **`AudioBuffer`:** the queue id and the item it was made for. **`PlayerQueueData`:** the
  record of buffers. Both runtime only.
* **One constant**, `STREAM_SLOT_RESERVATION`, next to the wait budgets.

## 7. Steps

Each ships alone. Sizes are source lines plus test lines and are meant to include the rig work
that made the earlier step of this work four times its estimate.

| Step | Source | Tests | Total |
|---|---|---|---|
| 1. The record | about 60 | about 200 | about 260 |
| 2. The cleanup | about 230 | about 650 | about 880 |
| 3. The slot | about 110 | about 350 | about 460 |
| 4. Deletions | about 110 out | about 150 | about 260 |
| 5. R10 | about 10 out | about 100 | about 110 |

### Step 1: a buffer knows its queue and its track

* **Contents.** `get_buffer` reports a new buffer to the queue; the queue records it and puts
  its id and the item on the buffer; both in the diagnostics. Recording a buffer drops the
  entries whose buffer was cleared. Nothing else reads them yet.
* **Changes no behaviour.**
* **New tests.** The record and the owner for every creator: a start, the preparation, a
  speaker's request, the flow stream and its prefetch, a seek that replaces the buffer, the
  target of a transfer; an entry whose buffer was cleared is gone at the next walk; a buffer of
  an item that left the list is still there.

### Step 2: audio that left a queue's scope is released at once

* **Contents.** Section 2: the scope, the function at the moments of 2.4, the three checks of
  2.5, a wait that is not a reader, `may_release` answered by the queue without the free-slot
  gate, complete buffers freed. A start keeps the track it tries in scope (2.2) and a
  replaced buffer is asked at that moment (2.4). The lobby preload of the quiz and the branch
  of `plays_now_or_next` for a queue without a session go. Every older path stays in place.
  It uses nothing of step 3.
* **What changes.** A buffer nobody reads goes when it leaves the scope, also on a provider
  with a free slot and also when it is complete; a track that left the queue is not fetched.
  Nothing that is read is touched that `dev` leaves alone.
* **Tests that pin what it changes.**
  * `tests/controllers/player_queues/test_stale_capture_slot.py`: the five that say a provider
    with a slot to spare keeps a capture (`:836`, `:857`, `:883`, `:905`, `:925`).
  * The two quiz tests there go: `test_a_track_the_quiz_buffered_in_its_lobby_keeps_its_audio`
    (`:729`), `test_a_staged_track_keeps_the_only_stream_until_its_queue_is_cleared` (`:755`),
    with the rig's `idle_queue` and `warm_first_quiz_track`
    (`tests/controllers/streams/stream_slot_rig.py:290`, `:308`).
  * `tests/providers/music_quiz/test_plugin.py`:
    `test_warm_first_track_stages_and_buffers_it_on_an_idle_target` (`:6170`) and
    `test_warm_first_track_stages_on_the_remote_virtual_player` (`:6221`) assert the staging
    only; `test_warm_first_track_keeps_the_staged_item_when_buffering_fails` (`:6270`) goes;
    `:6199`, `:6240` and `:6255` lose their assertion that nothing was buffered; the helper at
    `:6143` no longer installs `get_audio_buffer`.
  * `tests/controllers/streams/test_buffer_users.py`:
    `test_a_wait_for_the_first_audio_counts_as_a_user` (`:108`, a wait is not a reader),
    `test_a_source_that_delivered_everything_is_left_alone` (`:205`, complete buffers go too),
    `test_a_queue_that_is_not_playing_plays_next_the_item_it_is_on` (`:280`, that branch goes),
    `test_a_provider_with_a_slot_to_spare_keeps_every_source` (`:342`, the gate goes).
* **New tests.** A start that fails on two tracks with an ordinary audio error plays the
  third; an unread prepared buffer that is replaced goes at once, with a slot to spare and
  without a limit. Each of the four edits releases at the edit, filling and complete, with and
  without a limit; an edit that keeps the next track releases nothing; a track a stream reads
  or claimed survives the edit and goes when the stream ends; a replaced buffer survives its
  reader (P2); P3 and P11 open no source, and the preparation's wait ends at the edit; repeat
  one and each change of the sleep timer target release the prepared track, set after the
  preparation has finished; a queue without a session gets no audio; each moment and each of
  the three checks switched off in turn fails a test.
* **Lab.** On a per-item player and on a flow-mode player with crossfade, on the one-stream
  account: replace next, play next, remove, move, five edits in a row, each in the last 30
  seconds of a track; the same with repeat all on the last track; an edit after the player
  fetched the next track; skip, previous, seek inside and outside the buffer; repeat one and a
  sleep timer at the end of the track, switched on and off; a dynamic queue with an add in
  flow mode; a transfer, then an edit.

### Step 3: a speaker keeps its service across a track change

* **Contents.** Section 1: the queue on `acquire_stream_slot`, the reservation (after a
  source at playback pace, and only when the queue goes on with that provider), its early end
  in `_end_session`, `_held_stream_slots` on the provider's holders with today's readiness
  condition. It uses the owner of step 1 and nothing of step 2.
* **What changes.** Section 5, the first two points.
* **Tests that pin what it changes.**
  * `tests/models/test_music_provider.py`: `:112`, `:154`, `:174` stay as they are and have to
    pass on the new counter.
  * `tests/controllers/player_queues/test_stale_capture_slot.py::test_a_stale_capture_gives_its_slot_to_another_queue`
    (`:819`): the other queue now gets the slot after the reservation, not within its 2 s.
  * `tests/controllers/player_queues/test_paused_queue_slot.py` (26) and
    `tests/controllers/streams/test_stream_capacity.py` should pass unchanged; `:554`, `:593`
    and `:633` of the first are the ones that say who counts as a holder.
* **New tests.** P8, P9 and P10 as tests; the reservation lapses and the other queue starts
  within its budget; a stop, the pause timeout and a paused hand-over leave none; a second
  slot of a provider with two is not held back; a cancelled wait takes nothing; the queue's own
  request is never held by its own reservation. A preview that already waits gets the slot
  after a queue's last track and waits behind a reservation otherwise; a holder without audio
  yet is not refused behind. The counter itself: a grant and its reservation in one step, a
  waiter woken when a reservation lapses, a wait cancelled after it was woken, the reserving
  queue ahead of takers that waited longer.
* **Lab.** Two virtual players on the one-stream account at playback pace: the second asks
  again and again while the first plays an album through, skips, seeks and gets an edit; the
  first never loses a track. The second starts within the reservation after the first was
  stopped. From the debug log: the time between a source ending and the queue's next one
  asking, at a track boundary and at a skip, to settle the length.

### Step 4: delete what step 2 covers

* **Contents.** R5, R6, R7 and the 30 s guess of R2, each with the test of step 2 that fails
  when its replacement is switched off. The two READMEs.
* **Tests.** `tests/controllers/player_queues/test_stream_feeder.py`: `:365` (R6), `:525`,
  `:592`, `:621` (R5); `tests/controllers/streams/test_audio_buffer.py`: `:815`. Retargeted at
  what they protect.

### Step 5: a dynamic rebuild spares a track that is read

* **Contents.** R10 goes; the rebuild's `update_items` does the work.
* **Tests.** `test_dynamic_rebuild_keeps_tail.py::test_an_add_releases_the_audio_of_the_items_it_drops`,
  and three new ones (unread, read, claimed).

## 8. Risks and open choices

**Risks.**

1. **A start that lands in the reserved moment waits its 15 s out** when the first queue takes
   the slot back, and then gets the limit error, where a moment earlier it is refused at once
   by name (P7). If that is felt: the request first waits for the reservation to settle and
   then decides as today.
2. **The length is a guess until the lab has measured it.** Too short and P8 comes back in a
   slow lookup; too long and another speaker waits for nothing.
3. **The semaphore is replaced**, in one file, with the same bound. Its tests are few; the
   lab's two-player runs are the real check.
4. No code of the design exists. The probes measured `dev`.

**Open choices.** None. Decided since this version: a reservation only after a source at
playback pace; its length starts at 5 s and is corrected from the lab; all five steps are
built, R10 as the last.

## 9. What the reviews changed

**The maintainer's decisions since the first version.** The quiz lobby preload goes. The
admission of queues goes completely: a slot stays one open source, in the name of a queue and
a track, with a short reservation for the same queue; the limit keeps counting open sources.
The mixed-queue rule for holding a slot and the question what the default of five counts no
longer apply.

**Review 1** (eight findings, all accepted).

| Finding | Now |
|---|---|
| 1. Cached audio bypasses the playback limit; 5. a reload forgets the owner; 7. the semaphore does not prove a playback limit | gone with the admission: no playback limit is claimed |
| 2. Replacing a buffer can cut a reader | 2.2 rule 1, 2.6 |
| 3. Audio created after an edit | 2.5 |
| 4. Ending the session breaks an ended Sonos queue | no session is ended by the design |
| 6. Hooks miss repeat and the next track's details | 2.4; the slot has no hooks |
| 8. Steps not safe alone | section 7 |

**Review 2** (six findings).

| Finding | Now |
|---|---|
| 1. Queue state cannot tell whether a queue plays | gone with the admission; it is the reason the admission went |
| 2. The last check is not at the source; a wait keeps a buffer | 2.5 and "a wait is not a reader" in 2.2, measured as P11 |
| 3. A refusal can stop unrelated audio | gone with the admission: no refusal is added |
| 4. Sleep timers change what is next without a moment | 2.4 |
| 5. Previews and Ynison outside a playback limit | gone with the admission; they are takers without a queue (1.2) |
| 6. Step 2 depended on step 3 | 2.5 asks only for a session and the item; steps 2 and 3 use nothing of each other |

**Review 3** (sound with changes; four findings, all confirmed in the code and accepted).

| Finding | Now |
|---|---|
| 1. The check at the source turns a start's retry away | 2.2: the start moves `index_in_buffer` to the item it tries (step 2) |
| 2. An unused reservation can make a preview fail | 1.3: none is made when the queue has nothing to go on with on that provider; what a waiting preview sees is in 1.3 and section 5 (step 3) |
| 3. The holders as a fact add an immediate refusal | 1.5: the refusal keeps today's condition that the holder's audio is ready (step 3) |
| 4. A replaced buffer is not checked when it is replaced | 2.4: it is asked at that moment (step 2) |
| Tests missing from the inventory | section 7: `test_buffer_users.py:108`, `:205`, `:280`, `:342` in step 2, `test_stream_feeder.py:365` in step 4, the counter's tests in step 3 |
