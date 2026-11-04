# Builtin MCP server: technical plan

Status: decided with the maintainer on 2026-11-04. Story and epic: see the backlog issues linked
from the README.

## Summary

Music Assistant gets a small, builtin Model Context Protocol (MCP) server, written against the
existing aiohttp webserver with no new dependencies. Home Assistant finds it through the Supervisor
discovery that ESPHome's Device Builder already uses: one card, one click, no token. Other MCP
clients (Claude, ChatGPT, Codex, Cursor, Claude Code) connect to the same endpoint with a Music
Assistant token, and in a second phase through OAuth with PKCE so nothing has to be copied by hand.
The community `fastmcp_server` plugin stays as an extended alternative, renamed "MCP Server
(Community)".

## Verified facts (2026-11-04)

Home Assistant `mcp` integration (core `dev`, `homeassistant/components/mcp/`, `mcp==2.3.0`):

- Config flow sources: `user` (manual URL) and `hassio` (Supervisor discovery, core#180378). No
  zeroconf. The discovery payload is `{"service": "mcp", "config": {"url": "<url>"}}`; only `url`
  is read. The user sees one confirm card; on confirm HA runs `initialize` and requires
  `capabilities.tools`. The entry's unique id is the Supervisor discovery uuid, so an add-on
  uninstall or an explicit discovery delete removes the entry.
- No authentication on that path. A 401 diverts the flow into OAuth, where the user must paste a
  client id and client secret (Application Credentials). HA's OAuth client does no PKCE and no
  dynamic client registration.
- Transport: streamable HTTP first, SSE fallback only when the first POST gets 404 or 405. HA uses
  `initialize`, `tools/list` (every 30 minutes) and `tools/call`, with a fresh session per call.
  Handshake protocol version 2025-11-25. No prompts, no resources.
- `tools/call` is sent with the tool name and arguments only. The `llm_context` (which carries
  the HA user id) is available in the integration but never forwarded. The `mcp` SDK's
  `ClientSession.call_tool` has a `meta` parameter that lands in `params._meta`.

ESPHome Device Builder (`esphome/device-builder` #2733, #2736; `esphome/esphome` #19413;
`esphome/home-assistant-addon` #272):

- Hand-rolled JSON-RPC on aiohttp, no `mcp` runtime dependency. About 600 lines of transport and
  registry, 20 tools. Methods: `initialize`, `ping`, `tools/list`, `tools/call`. GET answers 405.
- Trust: the request must arrive on the ingress site, and the TCP peer must be loopback or the
  Supervisor. The public site sits behind the normal bearer middleware. (Music Assistant does
  not copy the Supervisor part: its ingress proxy relays signed-in HA users, see below.)
- Discovery URL `http://127.0.0.1:<ingress_port>/api/mcp`. The add-on `config.yaml` must list
  `mcp` under `discovery:`, or the Supervisor refuses the post with 403.

Music Assistant server:

- One aiohttp app on two sockets (`helpers/webserver.py`). Public port 8095. Internal port 8094
  only when running as add-on, bound to the first `172.30.32.x` address
  (`controllers/webserver/controller.py:404-414`). Dynamic routes serve on both.
- `is_request_from_ingress(request)` (`helpers/auth_middleware.py:557`) checks that the socket is
  the internal site. The HA system token is already refused anywhere else (`:146-149`). The add-on
  runs with `host_network: true`, like ESPHome.
- `DiscoveryController._announce_to_homeassistant` (`controllers/discovery/controller.py:482`)
  posts the `music_assistant` discovery at startup and every 24 hours through
  `helpers/hassio.supervisor_request`.
- Impersonation exists: `resolve_impersonated_user(mass, AuthProviderType.HOME_ASSISTANT, <ha user
  id>, required=False)` maps a HA user to its linked MA account; the service user
  `homeassistant_system` holds `users.impersonate`.
- `PendingLoginStore` and `verify_pkce` (`helpers/login_flow.py`) exist for the sign-in epic.
- Dependencies: the `mcp` SDK pulls 28 packages (pydantic, starlette, uvicorn, httpx, anyio, ...)
  even for its low-level server; `fastmcp==4.1.3` pulls 67. The community plugin is 12.5k lines
  with its own policy layer, ASGI bridge and connect wizard.

MCP spec: current revision 2026-07-28 is a new "modern" era (no `initialize`, `server/discover`).
HA's client and ESPHome's server speak the handshake era (2025-03-26 to 2025-11-25). Authorization
is optional; when used, the server must publish RFC 9728 protected resource metadata and the
authorization server RFC 8414 metadata. There is no network discovery in the spec; the Supervisor
route is the only one HA picks up without typing.

## Decisions

1. Own builtin server, not the community plugin. Zero new dependencies, hand-rolled stateless
   streamable HTTP like ESPHome.
2. A builtin plugin provider `providers/mcp_server/` (`type: plugin`, `builtin: true`,
   `allow_disable: true`), display name "Model Context Protocol Server (MCP)". It gets its config
   card in the plugin list. The HA discovery post stays in the `DiscoveryController`.
3. A fixed, hand-written tool set with compact results. No prompts, no resources, no policy layer:
   a tool runs as the request's user and the existing roles, scopes and filters apply.
4. Toward HA: Supervisor discovery `mcp` with the internal URL, no token on the trusted path,
   which is HA core reaching the internal socket from the host itself. A tool call from HA runs
   as the `homeassistant_system` service user, or as the HA user when HA sends its user id in
   `_meta` (small core PR); an unlinked HA user falls back to the service user, never
   auto-created. The Ingress proxy is not trusted this way; its requests carry the signed-in HA
   user and are authenticated as such.
5. OAuth (PKCE plus dynamic client registration) for external clients is phase 2 in its own PR.
   Until then external clients use a long-lived Music Assistant token as bearer.
6. The community plugin is renamed "MCP Server (Community)" with a description that says it is
   the extended server with its own options. Long term it may be removed; ownership marking
   follows the catalog epic (backlog#203).

## Architecture

### Transport (`providers/mcp_server/transport.py`)

- Route `POST /mcp` registered with `mass.webserver.register_dynamic_route`; `GET /mcp` and
  `DELETE /mcp` answer 405 with `Allow: POST`.
- `Content-Type` must be `application/json` (else 415; this forces a CORS preflight for browser
  callers). `Accept` must include `application/json` (else 406). Body is one JSON-RPC 2.0 message;
  a batch array is refused with 400.
- `MCP-Protocol-Version` header: absent means 2025-03-26; a value outside
  `{2025-03-26, 2025-06-18, 2025-11-25}` is 400.
- A notification or response gets 202 with an empty body. A request gets 200 with one JSON-RPC
  response object; JSON-RPC errors (`-32601` unknown method, `-32602` invalid params, `-32600`
  invalid request) are HTTP 200. No `Mcp-Session-Id`, no SSE: the server is stateless.
- Methods: `initialize` (returns the requested version when supported, else 2025-11-25;
  `capabilities: {"tools": {}}`; `serverInfo` name and version; a short `instructions` text for
  the model), `notifications/initialized` (202), `ping`, `tools/list`, `tools/call`.
- Origin: when an `Origin` header is present it must match one of the server's own origins
  (loopback, base URL, external URL), else 403. Skipped on the trusted internal path. Requests
  without `Origin` (server-side clients) pass.
- `set_current_peer_address(request.remote)` and `set_request_priority(NORMAL)` like `/api`.

### Trust and identity (`providers/mcp_server/auth.py`)

- Internal path: `is_request_from_ingress(request)` is true, the TCP peer is the host itself
  (peer address equals the local socket address, which is what HA core on the host network looks
  like), and the request path is `/mcp/<secret>`: a per-install secret the plugin generates once
  and stores, which only reaches Home Assistant core through the Supervisor discovery (other
  add-ons cannot list discoveries). Then no token is required and the request runs as the
  `homeassistant_system` user from `get_homeassistant_system_user`. The host-peer check alone
  would trust every process on the host network, including other `host_network` add-ons, and
  with `_meta` let them act as any linked user; the secret closes that at no cost to the one-click
  setup (decided 2026-11-05). `POST /mcp` without the secret is the bearer endpoint for everyone,
  the host included. The Supervisor (`HASSIO_SUPERVISOR_IP`) is deliberately not on
  that list: on this socket it is the Ingress proxy relaying signed-in HA frontend users, so its
  requests go through `get_authenticated_user` like any other and run as the ingress user from the
  `X-Remote-User-*` headers. Any other peer needs a bearer like everywhere else. Refusals log the
  peer at debug level so the guard can be verified on a real install.
- Everywhere else: `Authorization: Bearer <token>` through `get_authenticated_user(request)`.
  Missing or invalid: 401 with `WWW-Authenticate: Bearer realm="Music Assistant"`. Phase 2 adds
  `resource_metadata="..."` to that header.
- Impersonation: on the tokenless host path only, `tools/call` reads `params._meta["<key>"]` with the HA
  user id (key constant shared with the HA core PR, proposal `home-assistant.io/user_id`),
  resolves it with `resolve_impersonated_user(..., HOME_ASSISTANT, ..., required=False)` and sets
  `impersonated_user` for the call. `_meta` is ignored on the bearer path: there the token is the
  user. No tool ever takes a `user` argument.
- Each tool declares a required scope; the dispatcher checks `has_scope` before the call and
  `has_player_access` for the player it targets, so a restricted member sees and controls exactly
  what the API would allow.

### Tools (`providers/mcp_server/tools.py`)

A small decorator (`@mcp_tool(name, scope)`) registers an async function and derives the JSON
`inputSchema` from its signature: `str`, `int`, `float`, `bool`, `Literal[...]`, `list[str]`,
optional values with defaults, and the `:param:` lines of the docstring as property descriptions.
Descriptions are written for a language model (English, not translated). Results are compact
dictionaries: a media item is `{uri, name, media_type, artists, album, duration}` and never a raw
Music Assistant object. Errors come back as a tool result with `isError: true` and a one-line
message; a player lookup failure lists the available player names.

Players are addressed by id or name (case-insensitive, `players/get_by_name`). Media is addressed
by uri, or by a name that the tool resolves through search.

| tool | scope | arguments | what it does |
|---|---|---|---|
| `search_music` | library read | `query`, `media_type` (track, album, artist, playlist, radio, podcast, audiobook, any), `limit` | `music/search` over library and providers, compact rows |
| `get_players` | players read | none | every player the user may see: id, name, state, volume, muted, powered, group members, now playing |
| `play` | players control | `player`, `media` (uri or name), `media_type` hint, `mode` (replace, next, add, replace_next), `radio_mode` | resolves the item, `player_queues/play_media`, returns what was picked and where |
| `control_player` | players control | `player`, `command` (play, pause, play_pause, stop, next, previous, power_on, power_off, clear_queue, seek), `value` | the matching `players/cmd/*` |
| `set_volume` | players control | `player`, `level` (0-100) or `muted` | volume or mute |
| `set_playback_options` | players control | `player`, `shuffle`, `repeat` (off, one, all) | queue shuffle and repeat |
| `get_queue` | queues read | `player`, `limit` | current item and the next items |
| `get_recently_played` | library read | `limit` | `music/recently_played_items` |
| `get_favorites` | library read | `media_type`, `limit` | favourites |
| `add_favorite`, `remove_favorite` | library write | `uri` | favourite flag |
| `get_playlists` | library read | none | playlists the user may edit |
| `add_to_playlist` | library write | `playlist` (uri or name), `uris` | `music/playlists/add_playlist_tracks` |
| `group_players`, `ungroup_player` | players control | `player`, `members` | `players/cmd/set_members`, `players/cmd/ungroup` |

The exact set and wording are settled in the implementation PR; the table is the target.

### Provider (`providers/mcp_server/__init__.py`, `manifest.json`, `strings.json`)

- `type: plugin`, `builtin: true`, `allow_disable: true`, `multi_instance: false`, stage
  `stable`, codeowners `@music-assistant`. No requirements.
- Config entries: `expose_to_home_assistant` (bool, default true, only meaningful as add-on). Enable
  or disable of the plugin registers or removes the route.
- Discovery: when loaded as add-on with the option on, the `DiscoveryController` posts
  `{"service": "mcp", "config": {"url": "http://<HOSTNAME>:8094/mcp/<secret>"}}` next to the
  `music_assistant` discovery, at startup and on the daily cycle. The returned uuid is kept so a
  disable or unload sends `DELETE /discovery/<uuid>`, which removes the HA entry. A 403 from the
  Supervisor (add-on config not updated yet) is logged once at info level, never as an error.
- The community plugin's manifest gets name "MCP Server (Community)" and a description that says it
  is the extended, community maintained server with its own token policies and connect wizard.

### Tests (`tests/providers/mcp_server/`)

aiohttp test client against the real webserver: initialize and version negotiation, 400 on an
unsupported version, 405 on GET, 415 on a wrong content type, 202 for notifications, `tools/list`
schema shape, `tools/call` for a player command on a test player, 401 without a token on the public
site, tokenless access on the ingress site with a host peer, refusal with a foreign peer, `_meta`
impersonation setting the impersonated user and ignored on the bearer path, scope refusal for a
guest on a write tool, and the discovery payload including the delete on unload.

## Phases

0. Add-on repo: `- mcp` under `discovery:` in `music_assistant`, `music_assistant_beta`,
   `music_assistant_dev` and `music_assistant_nightly` `config.yaml`. Lands first.
1. Server: the builtin plugin, transport, tools, discovery, the `_meta` path, the community plugin
   rename, tests.
2. HA core: the `mcp` integration passes `meta={"<key>": llm_context.context.user_id}` on
   `call_tool` when a user id is present. A few lines in `coordinator.py`.
3. Server: OAuth authorization server for `/mcp`: `/.well-known/oauth-protected-resource/mcp`,
   `/.well-known/oauth-authorization-server`, `POST /oauth/register` (RFC 7591, public clients),
   `GET /oauth/authorize` (login through the existing sign-in, then a consent page),
   `POST /oauth/token` (authorization code with PKCE S256, optional refresh). Issued tokens are
   regular Music Assistant tokens named after the client, so they show up and can be revoked in
   the existing token list. Reuses `PendingLoginStore` and `verify_pkce`. The `WWW-Authenticate`
   header on 401 gets `resource_metadata`. The issuer is derived from the request origin, which
   ties into backlog#153.
4. Docs: a page on music-assistant.io for Home Assistant and for direct clients.

Later, not planned now: the MCP URL as a field in the `_mass._tcp` zeroconf record; an MCP server
card at `/.well-known/mcp/server-card.json` once that extension settles; the HA `music_assistant`
integration starting an `mcp` entry through integration discovery for installs without the
Supervisor (needs phase 3, plus a pre-registered client id and secret because HA's OAuth client has
no dynamic registration); the 2026-07-28 protocol era.

## Risks and open points

- The host-peer guard assumes HA core reaches the internal port with the host's own address on the
  `172.30.32.0/23` network. Verified by reading; to be confirmed on the test server before the PR
  leaves draft. Fallback: rely on the secret path alone, which is still only known to HA core.
- An impersonated account that an admin disabled must be refused, not silently replaced by the
  service user; only a missing link falls back (found in review, fixed in the server PR).
- The Supervisor refuses the `mcp` discovery until the add-on config carries it; installs that
  update the server before the add-on just see an info line.
- A voice satellite often has no user id in the HA context, so per-user behaviour through HA is
  partial by nature; the service-user fallback covers it.
- HA's OAuth client needs a client id and secret typed by the user; phase 3 serves dynamic
  registration for clients that support it, and HA without the Supervisor stays manual.
