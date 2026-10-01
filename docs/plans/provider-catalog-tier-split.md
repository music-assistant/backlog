# Catalog: initial tier split

Companion of the [catalog plan](provider-catalog.md), input for backlog#205. Produced from an audit of every provider manifest on 2026-10-02; the decisions were taken with the maintainer the same day. Activity data about individual codeowners was used for the decisions and is deliberately not published here.

## Proposed `tier` per provider (audit of 2026-10-02, dev @ 5b53a4b79)

Rule, decided 2026-10-02: `tier: core` requires `@music-assistant` in `codeowners`; everything else starts as `community`. Core members who want the team behind a provider they own add the org handle to its manifest in the same PR. Dormant codeowners stay listed; nothing changes for them in this pass.

| type | core | community |
|---|---|---|
| Music source | 8 | 51 |
| Player support | 14 | 13 |
| Plugin | 16 | 9 |
| Metadata | 7 | 3 |
| Audio analysis | 4 | 0 |
| **all** | **49** | **76** |

### core (49)

| domain | type | stage | codeowners | note |
|---|---|---|---|---|
| acoustid_lookup | Audio analysis | stable | @music-assistant |  |
| loudness_analysis | Audio analysis | (missing) | @music-assistant |  |
| smart_fades | Audio analysis | (missing) | @music-assistant |  |
| sonic_analysis | Audio analysis | (missing) | @music-assistant |  |
| coverartarchive | Metadata | stable | @music-assistant |  |
| fanarttv | Metadata | stable | @music-assistant |  |
| itunes_artwork | Metadata | stable | @music-assistant |  |
| lrclib | Metadata | stable | @music-assistant |  |
| musicbrainz | Metadata | stable | @music-assistant |  |
| theaudiodb | Metadata | stable | @music-assistant |  |
| wikipedia | Metadata | stable | @music-assistant |  |
| ambient_sounds | Music source | stable | @music-assistant |  |
| builtin | Music source | stable | @music-assistant |  |
| filesystem_local | Music source | stable | @music-assistant |  |
| overcast | Music source | beta | @music-assistant |  |
| qobuz | Music source | stable | @music-assistant |  |
| radiobrowser | Music source | stable | @music-assistant |  |
| spotify | Music source | stable | @music-assistant |  |
| tunein | Music source | stable | @music-assistant |  |
| airplay | Player support | stable | @music-assistant |  |
| chromecast | Player support | stable | @music-assistant |  |
| dlna | Player support | stable | @music-assistant |  |
| fully_kiosk | Player support | stable | @music-assistant |  |
| hass_players | Player support | stable | @music-assistant |  |
| sendspin | Player support | beta | @music-assistant |  |
| snapcast | Player support | unmaintained | @music-assistant |  |
| sonos | Player support | stable | @music-assistant |  |
| sonos_s1 | Player support | stable | @music-assistant |  |
| squeezelite | Player support | stable | @music-assistant |  |
| sync_group | Player support | stable | @music-assistant |  |
| universal_group | Player support | experimental | @music-assistant |  |
| universal_player | Player support | stable | @music-assistant |  |
| wiim | Player support | stable | @music-assistant |  |
| airplay_receiver | Plugin | alpha | @music-assistant |  |
| hass | Plugin | stable | @music-assistant |  |
| hue_entertainment | Plugin | beta | @music-assistant |  |
| lastfm_scrobble | Plugin | stable | @music-assistant |  |
| listenbrainz_scrobble | Plugin | stable | @music-assistant |  |
| milkdrop_visualizer | Plugin | experimental | @music-assistant |  |
| music_quiz | Plugin | beta | @music-assistant @TimoPtr |  |
| openai_compatible | Plugin | beta | @music-assistant |  |
| openai_tts | Plugin | alpha | @music-assistant |  |
| party | Plugin | stable | @music-assistant |  |
| profiler | Plugin | alpha | @music-assistant |  |
| radio_playlist | Plugin | beta | @music-assistant |  |
| recommendations | Plugin | stable | @music-assistant |  |
| sendspin_source | Plugin | alpha | @music-assistant |  |
| sonic_similarity | Plugin | beta | @music-assistant |  |
| spotify_connect | Plugin | beta | @music-assistant |  |

### community (76)

| domain | type | stage | codeowners | note |
|---|---|---|---|---|
| genius_lyrics | Metadata | stable | @robert-alfaro |  |
| lastfm_recommendations | Metadata | stable | @ozGav |  |
| playlist_metadata | Metadata | experimental | @dmoo500 |  |
| abc_radio_network | Music source | beta | @OzGav |  |
| apple_music | Music source | stable | @MarvinSchenkel |  |
| ard_audiothek | Music source | stable | @jfeil |  |
| audible | Music source | stable | @ztripez |  |
| audiobookshelf | Music source | stable | @fmunkes |  |
| bandcamp | Music source | stable | @ALERTua @teancom |  |
| bbc_sounds | Music source | stable | @kieranhogg |  |
| deezer | Music source | stable | @jdaberkow |  |
| digitally_incorporated | Music source | stable | @benklop |  |
| emby | Music source | stable | @hatharry |  |
| feiniu_music | Music source | experimental | @neqq3 |  |
| filesystem_google_drive | Music source | beta | @ozgav |  |
| filesystem_onedrive | Music source | beta | @ozgav |  |
| gpodder | Music source | stable | @fmunkes |  |
| ibroadcast | Music source | stable | @robsonke |  |
| iheartradio | Music source | beta | @OzGav |  |
| internet_archive | Music source | stable | @ozgav |  |
| itunes_podcasts | Music source | stable | @fmunkes |  |
| jellyfin | Music source | unmaintained | @lokiberra @Jc2k | drop @Jc2k (inactive since 2025-01), keep @lokiberra |
| kion_music | Music source | beta | @TrudenBoy |  |
| mammamiradio | Music source | alpha | @florianhorner |  |
| musicme | Music source | stable | @JulienDeveaux |  |
| neteasecloudmusic | Music source | stable | @xiasi0 |  |
| nicovideo | Music source | (missing) | @Shi-553 |  |
| nts | Music source | stable | @mike-sheppard |  |
| nugs | Music source | stable | @brian10048 |  |
| opensubsonic | Music source | stable | @khers |  |
| orf_radiothek | Music source | stable | @DButter |  |
| pandora | Music source | beta | @chrisuthe |  |
| phishin | Music source | stable | @ozgav |  |
| plex | Music source | beta | @anatosun |  |
| pocketcasts | Music source | stable | @yfhyou |  |
| podcast_index | Music source | stable | @ozgav |  |
| podcastfeed | Music source | stable | @saeugetier |  |
| qqmusic | Music source | stable | @xiasi0 |  |
| radioparadise | Music source | stable | @ozgav |  |
| rain_mood | Music source | experimental | @jlpouffier |  |
| siriusxm | Music source | stable | @btoconnor @MizterB |  |
| somafm | Music source | stable | @macegr |  |
| soundcloud | Music source | stable | @robsonke |  |
| storytel | Music source | beta | @jonasbp2011 |  |
| sverigesradio | Music source | (missing) | @romany |  |
| teddycloud | Music source | beta | @yoyixms |  |
| tidal | Music source | stable | @jozefKruszynski |  |
| vrt_max | Music source | experimental | @bollewolle |  |
| webdav | Music source | stable | @ozgav |  |
| yandex_music | Music source | stable | @TrudenBoy |  |
| yoto | Music source | beta | @pantsman0 |  |
| yousee | Music source | stable | @math625f |  |
| ytmusic | Music source | beta | @MarvinSchenkel |  |
| zvuk_music | Music source | stable | @TrudenBoy |  |
| alexa | Player support | experimental | @alams154 |  |
| amplipi | Player support | beta | @micro-nova |  |
| bluesound | Player support | stable | @cyanogenbot |  |
| bose_soundtouch | Player support | alpha | @Odn0 @fmunkes |  |
| heos | Player support | stable | @Tommatheussen |  |
| local_audio | Player support | deprecated | @iVolt1 |  |
| mpd | Player support | stable | @OzGav |  |
| msx_bridge | Player support | beta | @TrudenBoy |  |
| musiccast | Player support | stable | @fmunkes |  |
| raumfeld | Player support | experimental | @Simanias |  |
| roku_media_assistant | Player support | stable | @medievalapple |  |
| samsung_wam | Player support | beta | @oliver-stevens |  |
| yandex_station | Player support | beta | @trudenboy |  |
| ai_radio | Plugin | alpha | @swiftbird07 |  |
| ariacast_receiver | Plugin | alpha | @AirPlr |  |
| fastmcp_server | Plugin | experimental | @TrudenBoy |  |
| plex_connect | Plugin | beta | @anatosun |  |
| smart_playlist | Plugin | beta | @dmoo500 |  |
| subsonic_scrobble | Plugin | stable | @clusters |  |
| vban_receiver | Plugin | stable | @sprocket-9 |  |
| yandex_smarthome | Plugin | alpha | @trudenboy |  |
| yandex_ynison | Plugin | stable | @TrudenBoy |  |

### Same pass, housekeeping

- Manifests without `stage` (default to stable at load): loudness_analysis, nicovideo, smart_fades, sonic_analysis, sverigesradio
- Codeowner handles with inconsistent casing (normalize to the GitHub login, 14 manifests): bluesound, filesystem_google_drive, filesystem_onedrive, internet_archive, lastfm_recommendations, phishin, podcast_index, radioparadise, roku_media_assistant, samsung_wam, subsonic_scrobble, webdav, yandex_smarthome, yandex_station
- `documentation` missing in 15 manifests (list in the audit CSV); add the docs page URL where one exists
- jellyfin: drop `@Jc2k`, keep `@lokiberra`; stage stays unmaintained
- Ask every community codeowner in the PR whether they want a `support_url`
