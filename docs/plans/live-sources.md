# Live sources: a place of their own, and names that say what they are

Status: decided with the maintainer on 2026-11-20; the page on 2026-11-21. Epic: #571. Owner: Marcel van der Veldt. This
is the technical companion of the epic on the project board, written so it can be fed to an agent
for implementation.

Every `path:line` was checked on server `dev` at `85136fcbd`, frontend `main` at `26a9180c5` and
models `main` at `74c71c1`; the Sendspin lines on server `dev` at `26682cc18`, after
music-assistant/server#8010. A path that starts with `src/` is in the frontend, one that starts
with "models" in the models package; every other path is relative to `music_assistant/` in the
server. An independent review checked the steps against the code on 2026-11-20; what it found is
worked in.

## Problem

Audio that comes in live has no place of its own and no name of its own.

- A user of the nightly asked for a name of its own for the line-in of a Sendspin speaker: the
  input is the record player, the output is the living room. That name exists now ("Line-in"
  until it is set, `providers/sendspin/sources.py:831-838`, music-assistant/server#8010); every
  other live source still makes up its own name.
- Now playing can show the same text twice. The title falls back to the name of the source
  (`models/player_state.py:1197`), a source can send its own name as the stream title, as
  Sendspin does with the name of the input (`providers/sendspin/sources.py:245-247`), and the badge below the title shows the name of the same
  source (`src/composables/nowPlayingSource.ts:31-33`, `src/helpers/players.ts:275-282`).
- On the player a source belongs to, its name still repeats the name of that player: "Spotify
  Connect (Kitchen)" in the source list of Kitchen (`providers/spotify_connect/provider.py:911`,
  `providers/airplay_receiver/provider.py:524`).
- In Browse a live source sits between the music services, or in a folder named after the
  technique that delivers it, such as "Sendspin" (`controllers/music/controller.py:1085-1107`).
  The "Live Inputs" folder that once held them was removed in music-assistant/server#3964.
- Nowhere lists all of them. Receivers that the user adds are under Settings, Music with the
  label "Live sources" (`src/helpers/source_labels.ts:53`, `src/translations/en.json:1124`). The
  line-in of a speaker is in the settings of that speaker, and a device that is only an input is
  under Players, "Audio inputs" (`src/helpers/player_sections.ts:22`, `:53`).

## What exists

**Two kinds of source.**

| | Live source | Own source of a speaker |
|---|---|---|
| Examples | line-in of a Sendspin device, Spotify Connect, AirPlay Receiver, VBAN, AriaCast, Ynison | Line-in and TV on a Sonos, Optical and Bluetooth on a WiiM |
| Model | `AudioSource`, a media item (models `media_items/media_item.py:559-614`) | `PlayerSource` only (models `player.py:172-201`) |
| Plays on | any player: the audio streams into Music Assistant | that speaker, and for some systems their other speakers (below): Music Assistant only switches it |
| Delivered by | a provider with `AudioSourceMixin` (`models/media_capabilities.py:300-360`) | the player provider, in the player's source list (`providers/sonos/player.py:319-322`) |

**Live sources today.**

| Source | Name | Belongs to | Can be started from Music Assistant |
|---|---|---|---|
| Sendspin line-in | the name of the input on its own player, `"{input name} ({player name})"` elsewhere; `"{provider name} ({player name})"` for a device that is only an input (`providers/sendspin/sources.py:826-838`) | the device (`sources.py:204-218`) | yes (`sources.py:452`) |
| Spotify Connect | `"{provider name} ({player name})"` (`providers/spotify_connect/provider.py:911`) | one player each (`provider.py:328`) | yes (`provider.py:930`) |
| AirPlay Receiver | `"{provider name} ({player name})"` (`providers/airplay_receiver/provider.py:524`) | one player each (`provider.py:249`) | no (`provider.py:539`) |
| VBAN Receiver | `"{manifest name}: {stream name}"` (`providers/vban_receiver/provider.py:118`) | no player | yes (`provider.py:132`) |
| AriaCast Receiver | provider name (`providers/ariacast_receiver/__init__.py:151`) | no player | no (`__init__.py:166`) |
| Yandex Ynison | provider name (`providers/yandex_ynison/provider.py:243`) | no player | no |

A provider instance is named by the user in its settings (`models/provider.py:282-287`). A device
that is only an input is a player of type `SOURCE`, hidden by default
(`providers/sendspin/player.py:3528-3541`), and is shown as "Audio input".

**Inputs a system shares.** Some systems can play the input of one speaker on their other
speakers, without the audio passing through Music Assistant. Sonos loads the line-in of any
speaker of the household on a group (music-assistant/aiosonos `aiosonos/group.py:389-412`); the
server asks for the speaker's own only (`providers/sonos/player.py:804`). The TV input has no
such parameter: a speaker switches to its own (`aiosonos/api/namespaces/home_theater.py:21-35`),
and other speakers hear it by joining its group. HEOS plays an input of another device (pyheos
`player.py:451-456`); the server collects the inputs of all devices
(`providers/heos/provider.py:220-238`) and lists a player's own only
(`providers/heos/player.py:187-190`, `:550`).

Most of them also send a placeholder as the stream title until a track is known: the name of the
input for Sendspin (`sendspin/sources.py:245-247`), "AirPlay | name"
(`airplay_receiver/provider.py:552`), "Spotify Connect | name"
(`spotify_connect/provider.py:762`), "AriaCast Ready" (`ariacast_receiver/__init__.py:134`) and
the stream name for VBAN (`vban_receiver/provider.py:251-252`).

**Where a live source shows.**

- *Source list of a player.* The player collects the sources bound to it and the inputs of its
  own device (`models/player.py:2605-2613`); the state turns them into `PlayerSource` entries
  with `name=source.name` (`models/player_state.py:840-858`). While a source plays, the entry of
  its session is listed first and wins (`player_state.py:811-827`, `:837-838`). The app offers
  the entries that are not passive under "Select source"
  (`src/helpers/player_menu_items.ts:307-332`). The id of a live source in that list is its uri,
  `<provider instance>://audio_source/<id>` (models `helpers.py:135-137`).
- *Playing.* A live source is not queued: a request to play one selects it as the source of the
  player (`controllers/player_queues/controller.py:844-848`), which starts a session on that
  player (`controllers/players/audio_sources.py:900-902`). The session holds the source
  (`audio_sources.py:76-77`); a provider can hand it a rebuilt one (`:348-369`).
- *Members of a group and synced players.* They copy the media and the active source of the
  player they follow (`models/player_state.py:1048-1051`, `:722-724`), while their source list
  stays their own.
- *Browse.* The root lists every provider that delivers live sources: its one startable source
  directly, or a folder named after the provider (`controllers/music/controller.py:1085-1107`).
  A source of a provider that binds its sources to players is listed for its own player only;
  every other source, the line-in of a Sendspin device among them, is listed for every player
  (`controller.py:3134-3160`). A click plays (`src/helpers/media_item_actions.ts:158-161`). A
  list row has no second line for a live source (`src/components/ListviewItem.vue:238-241`), a
  card says "Audio source" (`src/components/discover/EditorialMediaCard.vue:288`), and neither
  has an icon for the type (`src/helpers/media_type_icon.ts:17-28`).
- *Now playing.* Title `metadata.title or session.source.name` (`models/player_state.py:1197`);
  the badge is the name of the active entry of the source list
  (`src/composables/externalSource.ts:22-36`), in the player bar and the fullscreen player
  (`src/layouts/default/PlayerOSD/PlayerTrackDetails.vue:118-123`, `:181-185`,
  `PlayerFullscreen.vue:107`). The app compares the album line with the source name
  (`src/composables/nowPlayingSource.ts:37-42`), not the title.
- *Other readers of the name.* The media a source starts on a player with
  (`controllers/players/audio_sources.py:909`), which some speakers show on their own display,
  and the names on its stream (`controllers/streams/controller.py:190`, `:326`, `:2312`).
- *Home Assistant.* The integration lists the names of the entries that are not passive and
  selects a source by its name (home-assistant/core
  `homeassistant/components/music_assistant/media_player.py:297-310`, `:518-525`). The names with
  the player in them are in the stable release too (`origin/stable`
  `providers/spotify_connect/provider.py:834`).
- *Search.* Live sources are not searched: the app asks for eight media types without them
  (`src/composables/useProgressiveSearch.ts:17-26`) and the result has no place for them.
- *Recently played.* Left out on purpose (`controllers/music/controller.py:2053-2056`).
- *Menu.* `MENU_ITEM_REGISTRY` (`src/components/navigation/utils/getMenuItems.ts:85`) holds
  Browse and Radio and nothing for live sources. An item with `available()` false is not shown
  at all (`getMenuItems.ts:76-80`, `:241`); users hide and reorder items through the
  `sidebar.menu` preference (`getMenuItems.ts:68`). The Radio page lists library radios only
  (`src/views/LibraryRadios.vue:101-111`).
- *Settings.* The Music page lists provider instances; a plugin that declares `audio_source`
  counts as a music source and gets the label "Live sources"
  (`src/helpers/settings_pages.ts:256-261`, `src/helpers/provider_config.ts:170-175`). A player
  provider that declares it, Sendspin, gets neither (`src/helpers/source_labels.test.ts:50`).

Radio and live sources already share one setting, "Default enqueue option for Radio and Live
Input item(s)" (`translations/en.json:882`, `src/layouts/default/ItemContextMenu.vue:1795-1797`).

**Built apart from this plan (#573).** A setting "Name of the input" on a Sendspin player that also
has an input, default "Line-in" (music-assistant/server#8010, key `source_name` in the config of
the device's Sendspin protocol, `providers/sendspin/player.py:553-563`), and a proposal for the
Sendspin spec so a device reports and keeps that name (Sendspin/spec#366, open). The Sendspin
provider hands out two versions of the source, a short one for the device's own list and a long
one for everything else (`providers/sendspin/sources.py:218`, `:826-829`). A test on
a running server showed where that stops: while the input plays on its own device the active
entry is the long form, because the entry of the session wins, and the title only follows a
rename at the next fresh start, because it was sent as stream metadata. Step 2 below moves the
choice between short and long to the core, which removes both.

## Concept

A **live source** is audio that plays live into Music Assistant and can be sent to any player.
It has a **name**, which says what it is, and it **belongs to** a player or to none.

- An input of a device belongs to the player of that device. Its name is what the user calls it
  ("Record player"), "Line-in" until then.
- A receiver that exists once per player (Spotify Connect, AirPlay Receiver) belongs to that
  player. Its name is the name of the receiver.
- A receiver that exists once (VBAN, AriaCast, Ynison) belongs to no player. Its name is the
  name the user gives it in its settings.

**One rule for showing it.** On the player it belongs to, a source shows by its name alone.
Where one line shows it elsewhere, it shows as "name (player)". A list with a second line shows
the name with the player on that line. A source that belongs to no player, or to a device that
is only an input, shows by its name alone everywhere.

| Where | Shown as |
|---|---|
| Source list of Living Room, and its Home Assistant entity | Record player · Spotify Connect |
| Now playing on Living Room, no track title reported | Record player |
| Browse | Record player (Living Room) · Spotify Connect (Kitchen) · VBAN Receiver |
| Live sources page, search, Settings | Record player, with Living Room on the second line · VBAN Receiver |
| Now playing on Kitchen, or on a group, no track title reported | Record player (Living Room) |
| Living Room as a member of that group | Record player |
| A device that is only an input, named "Record player" | Record player |

The **inputs of a speaker** (Line-in, TV, Optical) are not live sources: the speaker plays them
itself, Music Assistant only switches it. They keep the names the speaker gives them and stay
in its source list. Where a system can play the input of one speaker on its other speakers,
those speakers list it too, named by the same rule: "Line-in" on the speaker itself, "Line-in
(Den)" on the others. The Live sources page lists the inputs the selected speaker can switch
to in the same overview as the live sources.

## Decisions (2026-11-20)

1. Live sources get a menu item and a page of their own. Radio stays apart: stations are library
   items that are searched, sorted and favourited, live sources are a handful of devices and
   receivers in the home.
2. The page also lists the inputs the selected speaker can switch to, such as TV and Line-in
   on a Sonos. Inputs only: what else a speaker lists as a source, such as a music service of
   the device or a preset, stays in its source menu.
3. In Settings the inputs of devices are also listed on the Music page, under the label "Live
   sources"; a row opens the settings of its player. This replaces the line of
   music-assistant/backlog#519 that kept them under Players only.
4. The naming rule holds for every live source, Spotify Connect and AirPlay Receiver included.
   The names in Home Assistant change with it; that goes in the release notes.
5. Now playing never shows the same text twice: the badge is left out when it says what the
   title says.
6. A name of its own exists for the input of a Sendspin device (music-assistant/server#8010) and for a receiver
   through its settings. The inputs of a speaker keep the names the speaker gives them; no
   setting renames them.
7. An input that a system shares between its speakers is offered on those speakers: the
   line-in of a Sonos speaker on every other Sonos speaker and group, the inputs of a HEOS
   device on the other HEOS devices.
8. The design of the page is shown to the maintainer before it is built (mock-up of
   2026-11-20).
9. Browse gets one folder "Live sources". Search finds a live source by name. Not built: a row
   on Discover, and receivers that can only be started from a phone on the page. Recently played
   stays without live sources.

## Decisions (2026-11-21)

10. The page is one overview in the style of the other pages, such as Artists and Playlists: the
    live sources and the inputs of the selected speaker together. A row shows the name of the
    source, with the player that offers it on a second line ("Line-in", below it "Office"). This
    replaces the two groups of the mock-up of 2026-11-20.
11. Browse keeps one line per source, "Record player (Living Room)", because the mobile app and
    other clients show only the name. Browse can follow the page once they show a second line.

## Steps

### 1. Now playing shows a live source once (frontend, #572)

The badge is left out when the name of the source equals the title that is shown, the way the
album line is already compared (`src/composables/nowPlayingSource.ts:37-42`). The comparison is
on the name, not on the label, which can carry the protocol after the name
(`src/helpers/players.ts:275-282`). One place decides it, for the player bar and the fullscreen
player. This needs nothing from the server and fixes the report at once; with step 2 the title
and the badge also differ where they should.

### 2. One naming rule, applied by the server (models, server, #574)

A name that depends on where it is shown cannot be a text each provider builds for itself. The
provider says what the source is called and which player it belongs to; the core composes the
name at the moment it is read, for the place it is read for.

**models**
- `AudioSource` gets an optional field for the player the source belongs to, None by default.
  It holds the id the provider knows that player by, which may be the id of one of its
  protocols; the core resolves it to the player, as it does when it collects the standing
  sources (`models/player.py:2607-2612`). Pick a name that is neither `owner`, which the app
  reads as a subtitle on other items (`src/components/discover/EditorialMediaCard.vue:284`), nor
  `player_id` alone, which on a session is the player a source plays on
  (`controllers/players/audio_sources.py:76`).
- `PlayerSource` gets the same field, for an input a speaker offers from another speaker of its
  system (step 5), so the core composes its name by the same rule and the list of step 3 can
  show that speaker on a second line.

**Server: providers**
- Inside the server a source always carries its plain name.
- Sendspin: the name of the input, or the name of the device's player for a device that is only
  an input; the player is the one of the client (`providers/sendspin/sources.py:434-457`,
  `:849-852`). One version of the source for `get_audio_sources` and `get_player_input_sources`.
- Spotify Connect and AirPlay Receiver: the provider's name, and the bound player
  (`spotify_connect/provider.py:907-911`, `airplay_receiver/provider.py:520-524`).
- VBAN Receiver: the provider's name instead of the manifest name with the stream name
  (`vban_receiver/provider.py:118`), so it follows what the user calls it.
- A placeholder title that only repeats the name goes, so the core gives the title and a rename
  shows at once: Sendspin, AirPlay Receiver and Spotify Connect (lines above). "AriaCast Ready"
  and the VBAN stream name say something else and are for the builder to weigh.

**Server: core**
- One function in the players controller returns the name of a source as shown on a given
  player, or outside any player. No player in the name when the source belongs to none, or to a
  player of type `SOURCE`.
- The player asks it when it builds its state, so `player_state.py` stays a pure function: for
  its standing sources (`models/player.py:2605-2613`) and for the source of the session it plays.
  That session is its own, or the one of the group or the player it follows: a member does not
  copy a title that is the name of a source (`models/player_state.py:1048-1051`), it gets the
  name for itself.
- The other readers call the same function for the player the session plays on: the media a
  source starts with (`controllers/players/audio_sources.py:909`) and the names on its stream
  (`controllers/streams/controller.py:190`, `:326`, `:2312`).
- A source that is sent to a client as a media item carries the name for outside a player: in
  the folder of step 3 and in the answer of the item commands
  (`controllers/music/controller.py:1608-1619`). The list of step 3 carries the plain name and
  the player, for a second line. The session takes the source with its plain name.

**Not handled.** Two sources with the same name on one player: Home Assistant keeps the last
one (`media_player.py:300-307`). A player renamed while its source plays elsewhere shows under
its new name at the next update of that other player.

**Tests.** The rule per kind of source (bound, input, input-only device, no player); the source
list and the title on the own player, on another player, on a group and on the own player as a
member of that group; a rename of the input while the source plays.

**Release notes.** A source that Home Assistant listed as "Spotify Connect (Kitchen)" on the
Kitchen player is listed there as "Spotify Connect"; an automation that selects it by name needs
the new name. The title a speaker shows on its own display follows the same rule.

### 3. The list of live sources, and one folder in Browse (server, #575)

- A command that returns, for a player, the live sources a user can start: what the Browse root
  collects today (`controllers/music/controller.py:1088-1095`, `:3134-3160`). With them come the
  inputs that player can switch to: the entries of its source list that step 4 marks as an input
  and that are not passive, none for a player that is synced to another, as in the source menu
  (`src/helpers/player_menu_items.ts:311`). Each entry carries its plain name and the player it
  belongs to, so the app shows that player on a second line (decision 10). Without a player it
  returns the live sources for all players, which is what Settings and the menu need. A source
  that belongs to a player the user has no access to is left out; the new field makes that one
  check.
- The Browse root lists one folder "Live sources" in place of the entries per provider
  (`:1085-1107`), holding the live sources of that list, each with its name for outside a
  player, "Record player (Living Room)", so every client tells them apart on one line (decision
  11). It is left out when the list is empty. The folder
  needs a scheme of its own next to `discover` (`controllers/music/constants.py:72`), handled
  like that one with a way back to the root (`controller.py:1112-1131`), and a translated title.
- The listing per provider (`controller.py:1149-1158`) stays, so a path an older app holds keeps
  working.
- `tests/test_cross_type_features.py:866-922` covers the root and the listing per provider and
  changes with it. The MCP server filters browse paths by provider
  (`providers/fastmcp_server/target_filters.py:104`); the new scheme is added to what it lets
  through.

Older apps keep working: they show the folder like any other. An app that is newer than the
server has no command to call and leaves the menu item out.

### 4. Speakers say which of their sources are inputs (models, server, #576)

A source list mixes inputs with other things a speaker can switch to: Bluetooth and its network
mode on a WiiM (`providers/wiim/constants.py:49-60`), the music services of a HEOS device
(`providers/heos/provider.py:241-247`), presets on a Raumfeld (`providers/raumfeld/player.py:509`).
The page lists inputs, so an entry has to say that it is one.

- models: `PlayerSource` gets a flag for an input of the device, False by default.
- Server: the player providers set it where they build their entries for a physical input:
  Sonos and Sonos S1 (`providers/sonos/const.py:40-55`, `providers/sonos_s1/constants.py:42-50`),
  WiiM, HEOS (`providers/heos/player.py:193-199`), Bluesound
  (`providers/bluesound/player.py:478-490`), MusicCast, Bose SoundTouch, Samsung WAM, AmpliPi and
  the others that list one.
- The entry of a live source that is an input of a device (the line-in of a Sendspin device)
  carries it as well.

### 5. An input of one speaker on the other speakers of its system (server, #577)

- Sonos: a speaker lists the line-in of every other speaker of its household that has one, and
  selecting it loads that line-in on its group (`load_line_in` with the device,
  music-assistant/aiosonos `aiosonos/group.py:389-412`). TV stays with its own speaker.
- HEOS: a player lists the inputs of the other devices, which the provider already collects
  (`providers/heos/provider.py:220-238`), and plays one with the device it comes from (pyheos
  `player.py:451-456`).
- The names follow the rule of step 2 through the same function: "Line-in" on the speaker
  itself, "Line-in (Den)" on the others. The entries are marked as inputs (step 4), so the page
  and Home Assistant show them without a change of their own.
- Tested on real speakers before it is merged; Sonos S1 is for the builder to check.

### 6. The Live sources page and its menu item (frontend, #578)

- A menu item "Live sources" below Radio in `MENU_ITEM_REGISTRY`, with `available()` true when
  the list of step 3 is not empty or a player has an input to switch to. Hidden and
  reordered like every item; a new id takes its default place in a saved order by itself.
- The page is built like the other pages, such as Artists and Playlists, with their list and
  panel views. It shows the list of step 3 for the selected player in one overview: the live
  sources and the inputs that player can switch to, for a Sonos TV and Line-in, and with step 5
  the line-in of its other speakers. A row shows the name as its title and the player the source
  belongs to on a second line (`src/components/ListviewItem.vue:238-241`), by the current name
  of that player; a source without a player has none.
- A click on a live source plays it on the selected player, as in Browse
  (`src/helpers/media_item_actions.ts:158-161`); a click on an input selects it on that player.
  The context menu keeps "Play now" and gets "Settings" for administrators: the settings of the
  player a source belongs to, or of the provider for a source without one.
- The media type gets an icon (`src/helpers/media_type_icon.ts:17-28`). A live source in Browse
  keeps one line (decision 11).
- The app refreshes the list when players or providers come and go, and when the page opens.
- Built as decided with the maintainer on 2026-11-21; the built page is shown to him before it
  is merged.
- The page says which player it plays on, and what to do when none is selected.

Screenshots with made-up players, as for every UI change.

### 7. Inputs of devices on the Music page (frontend, #579)

The Music page in Settings lists, after the music sources, a row for every live source that is
an input of a device: its name, the label "Live sources", and the player it belongs to. A click
opens the settings of that player. The filter chip "Live sources" includes these rows. They have
no menu to remove or disable: an input exists because its device has it. The Marketplace is
unchanged.

### 8. Search finds a live source by name (frontend, #580)

The search window gets a section "Live sources" that matches the typed text against the names in
the list the app already holds. A click plays. Rows look like those of the page: the name, with
the player on a second line. The search of the server and of the music
services is not involved.

### 9. Docs (#581)

A page on music-assistant.io for live sources: what they are, where to find and start them, how
to name an input, and that Home Assistant selects them by name. On the `beta` branch.

## Order

Step 1 stands alone. The models fields of steps 2 and 4 are released before the server and the
app use them. Steps 3 and 5 need steps 2 and 4. Steps 6, 7 and 8 need the list of step 3.

## Later, and what this must not block

- **Several inputs on one device.** A source is named and attached to a player per source, not
  per device, so a second input is a second source.
- **Starting a live source on any player from Home Assistant** (music-assistant/backlog#376).
  The list of step 3 and the uri of a source are what the integration needs.
- **The name in the Sendspin spec.** Wherever the name is kept, the provider hands the core the
  plain name.
- **A page in the mobile app.** Until then it finds live sources in the Browse folder.

No stored data changes meaning: the Sendspin name is a new key, the receivers store player ids
only, and a new menu item joins a saved menu order by itself.
