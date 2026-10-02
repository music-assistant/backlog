# Sign in with your own identity provider: technical plan

Status: proposal 2026-10-01. Owner: Marcel van der Veldt. This document is the technical companion of the "Sign in with your own identity provider" epic (music-assistant/backlog#183) on the project board. It is written so it can be fed to an agent for implementation; every claim marked "verified" was checked in the referenced code (server at `27772e407`, its Home Assistant user mapping and Ingress resolution at `ca5a03f96`, models at `6626960`, frontend at `7a636358`, mobile at `a501ff18`, desktop at `912133f`, portal app.music-assistant.io at `35607a1`).

## Summary

Music Assistant (MA) keeps its own users and passwords. Households that already run an identity
provider (IdP) such as Authentik, Authelia, Keycloak or Pocket ID manage a second set of family
accounts, and the passkeys and two-factor rules of that IdP never reach MA. "Sign in with Home
Assistant" exists, but it builds its redirect from the server's LAN address, so it breaks on the
remote app and behind reverse proxies (support#6601, backlog#153).

This epic turns sign-in methods into providers of a new provider type `auth`. The MVP hardens the
sign-in flow first (a public URL for redirects worked out like Home Assistant's internal and
external URLs, PKCE, state and nonce with a lifetime, a one-time code instead of the login token in
the URL), then adds a generic OpenID Connect (OIDC) auth provider with presets, moves the Home
Assistant login onto the same flow as the `hass_auth` auth provider, and updates every client: the
web app (with a rebuilt login page), the remote portal, the mobile app, the desktop app and the
python client. Accounts gain an email address, linked identities keyed by issuer and subject, and
an admin policy that can switch password sign-in off once an admin has a linked identity.

The first follow-ups after the MVP are fully specified here: passkeys in MA's own sign-in, and
Sign in with Apple and Google through a small sign-in relay run by the Open Home Foundation (OHF).
Later items (session re-validation at the IdP, sign in on another device for Apple TV, invite
links, the future identity provider on the home operating system) are specified but not built.

The builtin password sign-in, join codes and guest access, and the Home Assistant ingress header
trust stay in the webserver core.

## Decisions taken with the maintainer (2026-10-01)

1. Phasing. The MVP is the hardened foundation plus generic OIDC, the Home Assistant login converged
   on it, and every app updated. Passkeys in MA's own sign-in and Sign in with Apple and Google
   through the OHF relay are the first follow-ups right after the MVP lands, fully specified now.
   Trusted device, kiosk mode and quick user switching are a separate story.
2. Foundation first: a public URL for redirects worked out like Home Assistant's (internal URL,
   external URL when set, remote app via app.music-assistant.io), PKCE, state and nonce with a
   lifetime, a one-time code instead of the JWT in the URL, linking rules with UI, and accounts that
   only sign in through an IdP.
3. A new provider type `auth` with its own base class. `oidc`, `apple`, `google` and the Home
   Assistant login are auth providers. The Home Assistant one depends on the `hass` plugin at
   runtime and is created automatically. The builtin provider (password sign-in, later passkeys) and
   join codes and guest access stay in the webserver core; ingress header trust stays core.
4. Remote app: a callback page hosted on app.music-assistant.io hands code and state to the app
   window, which completes the login over the data channel. Home Assistant on remote sessions uses
   `client_id = https://app.music-assistant.io` (fixes support#6601).
5. Apple and Google: an OHF-hosted sign-in relay (a small worker with a short-lived key-value store)
   holds OHF's Apple key and Google client, so every install gets both without registering
   anything. The relay stores nothing beyond the 10-minute state. MA verifies the upstream ID token
   by signature and nonce. This needs an OHF infra decision (hosting, secrets).
6. Linking: the identity key is issuer plus subject. Automatic linking only on a verified email that
   equals the email an admin set on the account, or by explicit linking from the signed-in profile.
   Never by username. Home Assistant keeps its current behaviour.
7. Provisioning: a per-IdP "create accounts for new users" switch, default off; default role
   `user`; group or role claim mapping applied at creation only. Home Assistant keeps
   `auth_allow_self_registration`.
8. Creating an account without a password requires an email in v1. Invite links (no email needed)
   are a later sub-issue.
9. Revocation in v1: MA session lifetimes apply; an account disabled at the IdP is refused at its
   next sign-in; admins disable accounts in MA. Re-validation through a refresh token is a later
   sub-issue.
10. Policy: "allow password sign-in" (default on) and "open the identity provider automatically"
    (default off). Password sign-in can only be switched off while an enabled admin has a linked IdP
    identity. `/?local=1` always shows the password form. When password sign-in is off, the server
    refuses password logins for everyone except admins (break-glass).
11. `Login.vue` is rebuilt in shadcn-vue inside the epic, with parity PRs first. The desktop app
    gets the system browser plus a `musicassistant-desktop://` deep link.
12. Pre-existing bugs found during research become separate neutral maintenance PRs now, outside
    the epic.
13. Board: a standalone epic (Ongoing, Area multi) on the board. Sub-issues are plain issues in the
    backlog repo, not board items. This document closes research issue #107; #93 gets a line.

## Requirements

- An admin adds the household's IdP under Settings > Sign-in methods, with presets for Authentik,
  Authelia, Keycloak, Pocket ID and Google (own client), and the login page shows "Sign in with
  <name>".
- Sign-in through an IdP works on the LAN address, on a configured external URL, on the remote app
  (app.music-assistant.io), in the mobile app and in the desktop app.
- The login token never travels in a URL on the new path. Old clients keep working unchanged.
- An IdP identity attaches to an existing account only through a verified email an admin set, or by
  the signed-in user linking it. A username match never links an OIDC, Apple or Google identity.
- New people get an account only when the admin enabled it for that IdP, with a default role and an
  optional group-to-role mapping applied once.
- Admins can create an account without a password (email required) and can switch password sign-in
  off once an admin has a linked identity. A configuration that would lock everyone out fails open.
- Sign in with Home Assistant becomes an auth provider on the same hardened flow and works on the
  remote app and behind a reverse proxy, with zero setup for Home Assistant users.
- Nothing in the epic waits for Home Assistant core to ship OIDC.
- Nothing blocks a later identity provider on the home operating system that also signs people in
  to third-party apps such as Immich.

## Architecture

Server paths below are relative to `music_assistant/` in the server repo unless a repo is named.

### Starting point (verified)

- `AuthenticationManager` (`controllers/webserver/auth.py:117`) owns `auth.db` at schema 5
  (`auth.py:73`). Tables `settings`, `users` (no email column), `user_auth_providers` with
  `UNIQUE(provider_type, provider_user_id)`, `auth_tokens`, `join_codes`, `roles`
  (`auth.py:1896-1984`). Foreign key cascades never fire because enforcement is off on the
  connection; the manual delete lists live in `delete_user` (`auth.py:1247-1251`) and
  `_prune_orphaned_user_rows` (`auth.py:2255-2269`) (verified).
- Login providers are plain classes in `controllers/webserver/helpers/auth_providers.py`:
  `LoginProvider` ABC (`:299`), `BuiltinLoginProvider` (`:365`) and `HomeAssistantOAuthProvider`
  (`:533`). The builtin provider stores the PBKDF2 hash as `provider_user_id` of a `builtin` row in
  `user_auth_providers` and verifies a password by looking that row up (`auth_providers.py:428-435`,
  `:476-477`) (verified).
- The Home Assistant provider keeps its state in a dict without a lifetime (`auth_providers.py:546`,
  `:607`), sends no PKCE, uses the redirect URI's origin as `client_id` (`:610`), guesses the Home
  Assistant URL from the redirect host when Home Assistant only reports the supervisor URL
  (`:587-602`) and silently links by username (`get_or_create_ha_user`, shared with the ingress
  path since server#6657) (verified). The auth manager registers it, not the `hass` plugin:
  `_setup_login_providers` (`auth.py:2086`) runs during webserver setup, before any provider is
  loaded, so the provider only appears through `_sync_ha_oauth_provider` (`auth.py:2117`), which
  `get_login_providers` calls on every listing (`auth.py:966-969`) (verified).
- Redirect path: `GET /auth/authorize` (`controllers/webserver/controller.py:1171`) returns a JSON
  `authorization_url` whose `redirect_uri` is `{base_url}/auth/callback?provider_id=...`
  (`auth.py:1106`). `GET /auth/callback` (`controller.py:1201`) mints a token (`:1229-1230`) and
  renders `helpers/resources/oauth_callback.html` with the JWT embedded (`:1250-1262`), which sends
  the browser to `return_url?code=<jwt>` via `build_code_redirect_url`
  (`helpers/redirect_validation.py:109`); external return URLs get a consent step
  (`controller.py:1238-1247`) (verified). The server's `/login` page (`controller.py:979-995`,
  `helpers/resources/login.html:193`) hands the web app `/?code=<jwt>` (verified).
- `base_url` "auto" resolves to the LAN publish IP (`controller.py:215-223`, `:624-626`).
  `external_url` exists (`controller.py:234-240`) but no auth code reads it (verified by grep).
- Redirect validation allows `musicassistant://` and three Home Assistant URLs
  (`redirect_validation.py:18-26`) and otherwise classifies same-origin, loopback, private IPs and
  `base_url` as trusted and every other http(s) URL as external (`:29-102`) (verified).
- Tokens are HS256 JWTs (`helpers/jwt_auth.py:39`) with a DB row; short-lived sessions slide to 30
  days with a 90-day cap (`auth.py:76-81`). No refresh tokens, no cookies (verified).
- Ingress: a second TCP site on the `172.30.32.x` address, port 8094 (`controller.py:364-375`,
  `constants.py:67`), recognized by socket address only (`helpers/auth_middleware.py:510-543`).
  HTTP requests and websockets resolve the ingress user through one helper, `resolve_ingress_user`
  in `auth_middleware.py` (server#6651), which maps the Home Assistant user with
  `get_or_create_ha_user`, the same helper the Home Assistant login uses (server#6657) (verified).
- Remote app: the WebRTC gateway bridges each data channel to
  `ws://localhost:8095/ws?webrtc_session_id=<id>` (`remote_access/gateway.py:144`, `:819`) and keeps
  its sessions in `WebRTCGateway.sessions` (`gateway.py:193`). The websocket handler reads that query
  parameter as is (`websocket_client.py:90`) (verified).
- Unauthenticated websocket connections receive no events: regular connections subscribe after the
  `auth` command (`websocket_client.py:173-177`, `:496`) (verified).
- Providers: `ProviderType` has `music, player, metadata, plugin, core, audio_analysis` and `unknown`
  with `_missing_` to UNKNOWN (models `enums.py:792-808`). `ProviderManifest` carries
  `multi_instance`, `builtin`, `allow_disable`, `depends_on` and `self_service` (default `true`)
  (models `provider.py:34-57`). The server's instance union is
  `ProviderInstanceType` (`models/__init__.py:21-23`) (verified).
- Dependency handling in `mass.py`: `_load_provider` returns silently while the dependency is not
  loaded (`mass.py:1436-1440`); after a provider loads, `load_provider_config` loads its enabled
  dependents (`mass.py:980-1027`); unloading a provider unloads its dependents (`mass.py:1163-1166`).
  Builtin providers get a default config at startup (`mass.py:1312-1319`) and load when enabled or
  not disableable (`mass.py:1324-1330`). The webserver and auth manager start before any provider
  loads (`mass.py:337`, `:371-376`) (verified).
- Models: `AuthProviderType {builtin, homeassistant}` with `_missing_` to BUILTIN (models
  `auth.py:28-37`). The impersonation parser guards against that fallback with an explicit
  membership check (`auth_middleware.py:340-353`). `API_SCHEMA_VERSION = 84`
  (`constants.py:51`) (verified).
- Setup flows: setup data is encrypted at rest (`controllers/config/flows.py:524`, `:790-795`);
  members with `config.providers.own` may start flows for `self_service` providers
  (`flows.py:158`, `:242`); the flow owner lives on `SetupFlowAccess` (`flows.py:64-71`), not on
  `SetupFlowContext` (`models/setup_flow.py:113-132`); `SetupSession.callback_url` is built on
  `base_url` (`models/setup_flow.py:186-188`) (verified).

### 1. The `auth` provider type

**Models (`music-assistant/models`).**

- `ProviderType.AUTH = "auth"` in `enums.py`, next to `AUDIO_ANALYSIS`.
- `AuthProviderType` gains `OIDC = "oidc"`, `APPLE = "apple"`, `GOOGLE = "google"` and
  `UNKNOWN = "unknown"`; `_missing_` returns `UNKNOWN` instead of `BUILTIN`. A builtin fallback lets
  an unknown identity read as a password account, which is the wrong failure. The server's
  impersonation parser (`auth_middleware.py:347`) keeps its membership check and additionally
  refuses `UNKNOWN`.
- `User` gains `email: str | None = None`, `has_password: bool = True` and
  `login_methods: list[str] = field(default_factory=list)`. `has_password` defaults to true so that
  data from a server that never sends the field reads as today. `login_methods` is filled by
  `auth/users` and `auth/user` only (values: `password`, `passkey`, and the sign-in method ids).
- New dataclasses in `auth.py`:

```python
@dataclass
class SignInMethod(DataClassORJSONMixin):
    provider_id: str          # "homeassistant" or an auth provider instance id
    provider_type: str        # plain str on purpose: an old peer must never coerce it
    name: str                 # button label
    icon: str                 # mdi name ("mdi-...") or a preset slug, never a URL
    requires_redirect: bool = True
    available: bool = True
    unavailable_reason: str | None = None   # translation key
    supports_remote_app: bool = False

@dataclass
class SignInOptions(DataClassORJSONMixin):
    methods: list[SignInMethod]
    password_enabled: bool = True
    auto_launch_provider_id: str | None = None

@dataclass
class AuthPolicy(DataClassORJSONMixin):
    password_login_enabled: bool = True
    auto_launch_provider_id: str | None = None
    can_disable_password_login: bool = False
    cannot_disable_reason: str | None = None   # translation key

@dataclass
class UserIdentity(DataClassORJSONMixin):
    identity_id: str
    user_id: str
    provider_type: AuthProviderType
    provider_id: str
    provider_name: str
    issuer: str
    email: str | None = None
    email_verified: bool = False
    display_name: str | None = None
    created_at: datetime = field(default_factory=lambda: datetime.now(UTC))
    last_login_at: datetime | None = None

@dataclass
class AuthorizationRequest(DataClassORJSONMixin):
    authorization_url: str
    state: str
    expires_at: datetime
```

- Release the models package and bump the pin in the server (sub-issue 1).
- Frontend: `ProviderType` in `src/plugins/api/interfaces.ts:509-515` gains `AUTH = "auth"`; the
  hand-written interfaces get the dataclasses above.

**Server base class (`models/auth_provider.py`).** `AuthProvider(Provider)`, following the
`audio_analysis_provider.py` precedent. Contract:

| member | meaning |
|---|---|
| `name` | button label (inherited `Provider.name`, `models/provider.py:199`; the oidc provider overrides it with its `button_label`) |
| `icon` | mdi name or preset slug, never a URL |
| `requires_redirect` | `True` for every v1 provider |
| `uses_pkce` | send PKCE S256 to the IdP (default `True`; `hass_auth` sets `False`) |
| `uses_nonce` | send and check a nonce (default `True`; `hass_auth` sets `False`) |
| `sign_in_available`, `unavailable_reason` | whether the method can be used right now, with a translation key when not |
| `supports_remote_app` | the method works through `https://app.music-assistant.io/auth/callback/` |
| `provisioning` | `ProvisioningSettings(create_users=False, default_role="user", role_mapping={})` |
| `async build_authorization_url(pending: PendingLogin) -> str` | the URL the browser opens |
| `async complete_authorization(pending: PendingLogin, params: Mapping[str, str]) -> ExternalIdentity` | turns the callback parameters into a verified identity; raises `LoginFlowError(translation_key)` |
| `async resolve_user(identity: ExternalIdentity, pending: PendingLogin) -> User` | default: `self.mass.webserver.auth.resolve_federated_user(identity, pending, self.provisioning)` |
| `get_sign_in_method() -> SignInMethod` | built by the base class from the members above |

The plan text calls the availability member `available`. `Provider.available` already exists as
the load state set by `mass.py` (`models/provider.py:59`), so the base class uses
`sign_in_available`; the wire field on `SignInMethod` stays `available`.

**Server wiring.**

- `AuthProvider` joins `ProviderInstanceType` (`models/__init__.py:21-23`); `mass.py` gets
  `is_auth_provider()` next to `is_audio_analysis_provider()` (`mass.py:149-153`).
- On load and unload of an auth provider, `mass.py` calls
  `self.webserver.auth.on_auth_provider_changed(instance_id, loaded)`, which drops that provider's
  pending logins on unload and fires `EventType.AUTH_SIGNIN_METHODS_UPDATED` (new, models).
- The auth manager keeps only the builtin provider in `login_providers` (`auth.py:128`) and lists
  methods from `mass.get_providers(ProviderType.AUTH)`. `LoginProvider`,
  `HomeAssistantOAuthProvider`, `_setup_login_providers`' Home Assistant branch and
  `_sync_ha_oauth_provider` are removed. `BuiltinLoginProvider` and `LoginRateLimiter` stay in
  `helpers/auth_providers.py` as core. `get_ha_user_details`, `get_ha_user_role` and
  `get_or_create_ha_user` stay where they are because the core ingress path uses them
  (`resolve_ingress_user` in `auth_middleware.py`, `controller.py:959`).
- `DEVELOPMENT.md` manifest table (`DEVELOPMENT.md:176`) lists `auth` as a type.

**Providers.**

| domain | manifest | config |
|---|---|---|
| `oidc` | `multi_instance: true`, `builtin: false`, `self_service: false` | setup flow (section 4) |
| `hass_auth` | see below | none (reads `auth_allow_self_registration`) |
| `apple` | `multi_instance: false`, `self_service: false` | none; added from the dialog (section 8) |
| `google` | `multi_instance: false`, `self_service: false` | none; added from the dialog (section 8) |

`hass_auth/manifest.json`:

```json
{
  "type": "auth",
  "domain": "hass_auth",
  "stage": "stable",
  "name": "Home Assistant",
  "description": "Sign in to Music Assistant with your Home Assistant account.",
  "codeowners": ["@music-assistant"],
  "documentation": "https://music-assistant.io/integration/sign-in/",
  "multi_instance": false,
  "builtin": true,
  "allow_disable": true,
  "depends_on": "hass",
  "self_service": false,
  "icon": "md:home-assistant",
  "requirements": []
}
```

Behaviour with the verified loader: the builtin config is created at every start
(`mass.py:1312-1319`); the builtin load returns silently while `hass` is not loaded
(`mass.py:1436-1440`); when `hass` loads, its enabled dependents load (`mass.py:980-1027`); when
`hass` unloads, `hass_auth` unloads (`mass.py:1163-1166`). Add-on users and `hass` plugin users keep
the button without doing anything, and an admin can disable it. `requirements` stays empty because
`hass_auth` never loads without `hass`, whose manifest installs `hass-client`
(`providers/hass/manifest.json:13`). In safe mode `hass` is not loaded, so there is no Home
Assistant button; password sign-in remains.

The sign-in method id of `hass_auth` stays `homeassistant`, the id the frontend, `login.html` and the
mobile app use today (`Login.vue:608`, `login.html:137`, `AuthenticationPanel.kt:110`). Its
identities stay in `user_auth_providers` with `AuthProviderType.HOME_ASSISTANT`, which ingress also
writes. `auth_allow_self_registration` stays a webserver config entry
(`controller.py:640-645`); `hass_auth` reads it. No migration.

### 2. Sign-in flow foundation (server)

New module `controllers/webserver/helpers/login_flow.py`.

**Transport.**

```python
class AuthTransport(StrEnum):
    DIRECT = "direct"     # browser or app talks to the server over HTTP(S)
    REMOTE = "remote"     # remote app over the WebRTC data channel
    INGRESS = "ingress"   # Home Assistant ingress (already signed in)

current_auth_transport: ContextVar[AuthTransport]
```

Set per websocket command and per HTTP request. REMOTE only when the peer address is loopback and
the connection's `webrtc_session_id` is a key of `remote_access.gateway.sessions`
(`gateway.py:193`); the bare query parameter (`websocket_client.py:90`) is never trusted on its
own. INGRESS from `is_request_from_ingress` (`auth_middleware.py:510`).

**Pending logins.**

```python
@dataclass
class PendingLogin:
    state: str                          # "w.<token>" (web) or "n.<token>" (native app)
    provider_id: str                    # auth provider instance id
    purpose: Literal["login", "link", "test"]
    transport: AuthTransport
    redirect_uri: str                   # exactly what was sent to the IdP
    redirect_target: Literal["server", "app"]
    expires_at: float                   # monotonic
    link_user_id: str | None = None     # purpose link: the user who started it
    idp_code_verifier: str | None = None
    nonce: str | None = None
    return_url: str | None = None       # validated
    return_url_category: Literal["trusted", "external"] | None = None
    client_code_challenge: str | None = None   # S256 only
    client_state: str | None = None
    device_name: str | None = None
    provider_data: dict[str, Any] = field(default_factory=dict)
```

`PendingLoginStore` lives in memory on the auth manager: TTL 600 s, `pop()` is single use, at most
512 entries (the oldest is evicted), `drop_provider(instance_id)`. A restart drops every pending
login, which only costs a retry.

`AuthorizationCodeStore` holds MA's own one-time codes: the key is `sha256(code)`, TTL 60 s, single
use, and each entry carries `state`, `user_id`, `purpose`, `client_code_challenge`, `device_name`
and, for purpose link, the identity id. A code is 32 bytes from `secrets.token_urlsafe`.

`ExternalIdentity` is what a provider hands to the resolver:

```python
@dataclass
class ExternalIdentity:
    provider_type: AuthProviderType
    issuer: str
    subject: str
    email: str | None = None
    email_verified: bool = False
    username: str | None = None
    display_name: str | None = None
    avatar_url: str | None = None       # https only
    groups: list[str] = field(default_factory=list)
    claims: dict[str, Any] = field(default_factory=dict)   # snapshot, size-capped
```

States carry a non-secret prefix: `w.` for the web, `n.` when the `return_url` uses the mobile app
scheme `musicassistant://`. The portal callback page forwards `n.` states to the app (section 6).

**Callback base (`WebserverController.get_auth_callback_base`).** Modelled on Home Assistant's
internal and external URL split. Inputs: the transport and the origin of the current request (the
`Origin` header of the HTTP request or websocket upgrade, else scheme plus `Host`).

| transport | callback base | callback URL |
|---|---|---|
| INGRESS | none (the user is already signed in; redirect methods are not offered) | none |
| REMOTE | `https://app.music-assistant.io` | `https://app.music-assistant.io/auth/callback/` |
| DIRECT, origin host known | the request origin | `<origin>/auth/callback` |
| DIRECT, otherwise | `external_url` when set, else `base_url` | `<base>/auth/callback` |

"Known" means loopback, one of `publish_addresses` (`controller.py:621-623`), the host of
`base_url` or the host of `external_url`. An unknown origin is never echoed into a redirect URI.
This also covers hairpin NAT: a LAN browser on the LAN address gets the LAN callback even when
`external_url` is set but unreachable from inside.

There is one fixed callback path, `/auth/callback`, without a `provider_id` query; the provider is
looked up from `state`. A `provider_id` query on an incoming callback is ignored.

Home Assistant specifics in `hass_auth`: `client_id` is `https://app.music-assistant.io` for the
app target and the origin of the redirect URI otherwise. Home Assistant's IndieAuth accepts a
redirect URI with the same scheme and host as `client_id` without fetching the client page
(`homeassistant/components/auth/indieauth.py` on the home-assistant/core dev branch, read during
research; verified). The Home Assistant URL comes from Home Assistant's
`network/url` (external, then cloud, then internal), as `_get_external_ha_url` already does
(`auth_providers.py:735-752`), else from the configured `hass` URL (`providers/hass/__init__.py:183-188`)
when it is not the supervisor URL, else `<scheme>://<host of MA external_url or base_url>:8123`
with a warning. Never from the redirect host. The token call needs no `code_verifier`:
`hass_client.utils.get_token(hass_url, code, client_id, grant_type)` sends none
(`hass_client/utils.py:80-81`, hass-client 1.3.1, verified).

**Start.** `auth/authorization_url` (websocket, `authenticated=False`) and `GET /auth/authorize`
gain arguments; the response gains fields. Old callers pass only the first two.

```text
auth/authorization_url(
    provider_id: str,                         # "homeassistant" or an auth provider instance id
    return_url: str | None = None,
    code_challenge: str | None = None,        # base64url SHA-256 of the client verifier
    code_challenge_method: str | None = None, # "S256" only; anything else is refused
    client_state: str | None = None,          # echoed back to return_url, max 256 chars
    redirect_target: str = "server",          # "server" | "app" (REMOTE transport only)
    device_name: str | None = None,
    purpose: str = "login",                   # "login" | "link" (link needs an authenticated connection)
) -> {"authorization_url": str, "state": str, "expires_at": str}
```

On failure the response keeps today's shape `{"authorization_url": None, "error": ...}`
(`auth.py:1083-1087`). The websocket variant validates `return_url` exactly as the HTTP variant does
(`controller.py:1184-1188`); `is_allowed_redirect_url` takes an aiohttp request today
(`redirect_validation.py:29`), so it gets a sibling that takes the origin host. `redirect_target
"app"` requires a client challenge. `purpose "test"` is internal to the setup flow.

**Callback.** `GET /auth/callback`, plus `POST /auth/callback` for `response_mode=form_post`:

1. Pop the pending login by `state`. Unknown or expired: an error page with a link back to `/`.
2. IdP `error`: redirect to `return_url?error=<code>&error_description=<text>&state=<client_state>`
   when there is a return URL, else the error page.
3. Otherwise `complete_authorization`, then by purpose: `login` resolves the user, `link` attaches
   the identity to `link_user_id`, `test` hands the identity to the waiting setup flow.
4. With a client challenge: mint a one-time code into `AuthorizationCodeStore` and 302 to
   `return_url?code=<one-time>&state=<client_state>`. Without a return URL: a small page that posts
   `{type: "ma-auth-code", code, state}` to a same-origin opener and closes.
5. Without a client challenge: today's behaviour byte for byte (JWT minted, `oauth_callback.html`,
   `?code=<jwt>`, consent step for external targets). Logged at debug as a legacy hand-back so its
   use can be measured before deprecating it.

`POST /auth/login` (`controller.py:997`) and `POST /setup` (`controller.py:1295`) with a
`return_url` follow the same rule: when the body carries `code_challenge`, `redirect_to` carries a
one-time code instead of the JWT. `GET /login` without `return_url` redirects to `/`;
`login.html` stays only for external clients that pass a `return_url`.

**Completion.** One command for every case: websocket `auth/exchange` (`authenticated=False`) and
`POST /auth/token` (same semantics, CORS headers like `/auth/login`).

```text
auth/exchange(state: str, code: str, code_verifier: str, device_name: str | None = None)
-> login: {"success": true, "access_token": str, "user": {...}}   # same user shape as auth/login (auth.py:1046-1054)
-> link:  {"success": true, "identity": UserIdentity}
-> fail:  {"success": false, "error": str, "translation_key": str}
```

Resolution by `state`: an open pending login with `redirect_target "app"` means `code` is the IdP
code; the server checks `S256(code_verifier) == client_code_challenge`, completes the authorization
server-side with its own IdP verifier and mints the token. Otherwise `code` must be an entry in
`AuthorizationCodeStore` with the same `state`, checked against the same challenge. Purpose `link`
requires the connection's user to be `link_user_id`. Failed exchanges count in a `LoginRateLimiter`
(`auth_providers.py:125`) keyed per connection like join codes (`auth.py:2582-2593`).

**Backward-compatibility rule.** A request without `code_challenge` behaves as today. New fields are
additive. Old in-flight logins die with the restart that installs the update, which only costs a
retry.

### 3. Accounts, identities and policy (server)

**`auth.db` v6.** `DB_SCHEMA_VERSION = 6`. Fresh installs get the column and table from
`_create_database_tables` (`auth.py:1896`); existing installs from `_migrate_database`
(`auth.py:2006`) under `if from_version < 6:`.

```sql
ALTER TABLE users ADD COLUMN email TEXT;   -- suppressed OperationalError, like v3 (auth.py:2028-2037)

CREATE TABLE IF NOT EXISTS user_identities (
    identity_id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    provider_type TEXT NOT NULL,
    provider_id TEXT NOT NULL,
    issuer TEXT NOT NULL,
    subject TEXT NOT NULL,
    email TEXT,
    email_verified INTEGER NOT NULL DEFAULT 0,
    display_name TEXT,
    username_hint TEXT,
    claims json NOT NULL DEFAULT '{}',
    created_at TEXT NOT NULL,
    last_login_at TEXT,
    refresh_token TEXT,
    UNIQUE(issuer, subject)
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_users_email ON users(email) WHERE email IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_user_identities_user ON user_identities(user_id);
```

Migration rule: idempotent, survives a half-applied previous run, never raises. The column add is
wrapped in `contextlib.suppress(OperationalError)`; index creation in `_create_database_indexes`
(`auth.py:1986`) is wrapped too. No data is rewritten. `user_identities` joins the delete list in
`delete_user` (`auth.py:1249`) and the prune list in `_prune_orphaned_user_rows`
(`auth.py:2260`). Home Assistant links stay in `user_auth_providers`.

Email: stored stripped and lower-cased, unique when set. Only a caller with `users.manage` may set
or change it (`auth/user/create`, `auth/user/update`); a self-edit is refused, so nobody can
pre-claim an address that would later auto-link an IdP identity.

**Resolver `AuthenticationManager.resolve_federated_user(identity, pending, provisioning)`.**

1. Look up `user_identities` by `(issuer, subject)`. Found: the user must be enabled, else
   `login_account_disabled`. Update the identity snapshot (`email`, `email_verified`,
   `display_name`, `username_hint`, `claims`, `last_login_at`) and return the user.
2. Purpose `link`: refuse with `login_identity_linked_elsewhere` when step 1 found another user;
   otherwise insert the identity for `link_user_id` and return that user.
3. `identity.email_verified` and the normalized email equals `users.email` of exactly one enabled
   user: insert the identity for that user and return it.
4. A username never links.
5. `provisioning.create_users`: create a user. Username from `identity.username`, else the email
   local part, else `user`, run through `normalize_username` (`auth_providers.py:40`) and
   de-duplicated with a `-2`, `-3` suffix. Role: the first `role_mapping` entry whose group is in
   `identity.groups`, else `default_role`; the role must exist (`_ensure_role_exists`,
   `auth.py:2211`) and is never `service`. Display name and avatar from the identity. Email set only
   when verified and not taken. Insert the identity. Mapping is applied at creation only.
6. Otherwise fail with `login_no_account`.

Profiles are never rewritten on later logins; only the identity snapshot is. `hass_auth` overrides
`resolve_user` and keeps today's behaviour (link by HA user id, then by username, then
self-registration with the HA admin role mapping, profile refresh from HA): it calls
`get_or_create_ha_user(..., allow_create=<auth_allow_self_registration>)` and refuses a disabled
user with `login_account_disabled` ("User account is disabled", as the Home Assistant login answers
today) and fails with `login_no_account` when it returns no user.

**Accounts and identities API.**

| command | scope | change |
|---|---|---|
| `auth/user/create` | `users.manage` | `password: str \| None = None`, `email: str \| None = None`; no password requires an email (`auth.py:1170-1222` today requires a password) |
| `auth/user/update` | self or `users.manage` | `email` (`users.manage` only), `current_password` required for a self change when the user has a password |
| `auth/users`, `auth/user` | `users.read` | `User.email`, `has_password`, `login_methods` |
| `auth/user/identities(user_id=None)` | self, or `users.read` for others | `list[UserIdentity]`, Home Assistant links mapped into the same shape |
| `auth/user/identity/unlink(identity_id, user_id=None)` | self, or `users.manage` | refused for a self unlink that removes the last way in (no password, identity or passkey left); an admin may, and the result says so |
| `auth/providers` | none | kept; entries gain additive `name` and `icon` |
| `auth/user/providers`, `auth/user/unlink_provider` | as today | kept as aliases (`alias=True`, `helpers/api.py:151-157`); the `provider_id` argument stays accepted |

`has_password` is the existence of the user's `builtin` row in `user_auth_providers`. The current
password check reuses `BuiltinLoginProvider.change_password`, which already verifies the old hash
(`auth_providers.py:481-504`) but is not called today: `_update_profile_password` always calls
`reset_password` (`auth.py:2360-2385`) (verified). Admin resets of other users do not ask for it.

**Policy.** Two rows in the existing `settings` table (`auth.py:1901-1907`), no schema change:
`policy.password_login_enabled` (bool, default true) and `policy.auto_launch_provider_id` (str,
default none).

```text
auth/policy() -> AuthPolicy                                   # users.read
auth/policy/update(password_login_enabled: bool | None = None,
                   auto_launch_provider_id: str | None = None) -> AuthPolicy   # users.manage
```

Guard: password sign-in can be switched off only while an enabled user with role `admin` has an
identity on a loaded, `sign_in_available`, redirect auth provider (after sub-issue 13 a passkey
counts too). Otherwise `cannot_disable_reason = "policy_no_admin_identity"`. The auto-launch provider
must be a loaded redirect method.

Fail-open at runtime: the effective value is `stored_value or not guard_satisfied()`. When an admin
identity becomes unusable (provider removed or unavailable, admin disabled), password sign-in is
effectively on again and a warning is logged once per change.

Enforcement lives in `BuiltinLoginProvider.authenticate` (`auth_providers.py:389`), so it covers
`auth/login` and `POST /auth/login`: when the effective value is off, a correct password for a
non-admin fails with `password_login_disabled`; admins pass (break-glass).

**`auth/signin_methods`.** Websocket (`authenticated=False`) and `GET /auth/signin_methods`, both
returning `SignInOptions`. INGRESS returns an empty method list. REMOTE marks methods without
`supports_remote_app` unavailable with `method_not_on_remote_app`. `EventType.AUTH_SIGNIN_METHODS_UPDATED`
fires when a method appears, disappears or changes availability; it reaches authenticated clients
only (see "Startup gap" under risks).

`API_SCHEMA_VERSION` goes from 84 to 85 in the last foundation PR (sub-issue 3).

### 4. OIDC auth provider (server, `providers/oidc/`)

Files: `__init__.py` (the provider), `client.py` (discovery, JWKS, token, ID token, userinfo;
aiohttp through `mass.http_session` and PyJWT, unit-testable without MA), `setup_flow.py`,
`presets.py`, `constants.py`, `strings.json`, `icon.svg`, `manifest.json` (`type auth`,
`multi_instance true`, `self_service false`, `requirements []`). PyJWT with crypto and
`cryptography` are already server dependencies (`pyproject.toml:24`, `:50`).

Load: discovery and JWKS are fetched at load. An unreachable IdP does not fail the load; the method
reports `sign_in_available = False`, `unavailable_reason = "idp_unreachable"`, and the next
`auth/signin_methods` call retries at most once per 60 s.

**Setup flow (`setup_flow.py`).** Setup data (encrypted at rest): `issuer`, `client_id`,
`client_secret`, `token_endpoint_auth_method`, `verify_ssl`, `allow_http`. Options (editable
later): `button_label`, `scopes`, `username_claim`, `display_name_claim`, `groups_claim`,
`create_users`, `default_role`, `role_mapping`.

1. `preset`: generic, Authentik, Authelia, Keycloak, Pocket ID, Google (own client). A preset fills
   defaults and help text (Keycloak `https://<host>/realms/<realm>`, Authentik
   `https://<host>/application/o/<slug>/`, Google `https://accounts.google.com`).
2. `issuer`: fetch `<issuer>/.well-known/openid-configuration` (10 s timeout). Errors:
   `issuer_unreachable`, `issuer_invalid_document`, `issuer_mismatch` (the document's `issuer` must
   equal the entered one exactly, after trimming one trailing slash from the input),
   `issuer_missing_endpoints` (`authorization_endpoint`, `token_endpoint`, `jwks_uri`),
   `issuer_no_code_flow` (`response_types_supported` lacks `code`), `issuer_insecure` (http unless
   the host is loopback or private and `allow_http` is ticked), `issuer_jwks_failed`,
   `issuer_already_configured` (another `oidc` instance has this issuer). Missing
   `code_challenge_methods_supported` is noted, not refused: PKCE is sent anyway.
3. `client`: read-only list of every redirect URI to register at the IdP: the public callback
   (`<external_url>/auth/callback`, when set), the LAN callback (`<base_url>/auth/callback`) and
   `https://app.music-assistant.io/auth/callback/` (trailing slash). Fields `client_id`
   (`client_id_required`), `client_secret` (optional, public clients use `none`). The token
   endpoint auth method is chosen from `token_endpoint_auth_methods_supported`:
   `client_secret_basic`, then `client_secret_post` when a secret is set, else `none`
   (`client_auth_unsupported` when nothing fits). The Google preset requires `external_url`
   (`google_requires_external_url`): Google only accepts https redirect URIs on a public domain.
4. `claims`: scopes (default `openid profile email`, plus `groups` for presets that use it),
   username claim (`preferred_username`), display name claim (`name`), groups claim (`groups`),
   button label (default: preset name, else the issuer host).
5. `provisioning`: `create_users` (off), `default_role` (`user`), `role_mapping` as `group=role`
   lines (`role_mapping_invalid`, `role_unknown`; `service` refused).
6. Optional `test`: a sign-in with purpose `test` through the normal `/auth/callback`, driven by
   `session.external_until(awaitable)` (`models/setup_flow.py:264`), because the IdP only knows the
   registered auth callbacks and not the setup flow's own callback (`models/setup_flow.py:186-188`).
   The step shows the received claims and offers "Link to my account", which needs the flow owner
   on `SetupFlowContext`: add `owner_user_id: str | None = None` there, filled from
   `SetupFlowAccess.owner_user_id` (`flows.py:70-71`). Errors from the test are shown, not fatal.

**Login (`complete_authorization` and friends).**

1. Authorization URL from the cached discovery document: `response_type=code`, `client_id`,
   `redirect_uri`, `scope`, `state`, `nonce` (32 bytes), `code_challenge` (S256 of the IdP
   verifier), `code_challenge_method=S256`.
2. Callback: when an `iss` parameter is present it must equal the issuer (RFC 9207); when the
   discovery document sets `authorization_response_iss_parameter_supported` it must be present
   (`idp_iss_mismatch`). An `error` parameter becomes `idp_error`.
3. Token request to `token_endpoint`: `grant_type=authorization_code`, `code`, the pending
   `redirect_uri`, `code_verifier`, client authentication per the chosen method. Non-200 or no
   `id_token`: `token_exchange_failed`.
4. ID token checks, every one required:
   - header `alg` is in `id_token_signing_alg_values_supported` (default `RS256`), never `none`;
     `HS256/384/512` only with a client secret, verified with that secret;
   - signature against the cached JWKS; an unknown `kid` refetches the JWKS once (at most once per
     minute); a failed fetch keeps the last good set;
   - `iss` equals the issuer;
   - `aud` contains `client_id`; with more than one audience, `azp` equals `client_id`;
   - `exp` in the future and `iat` not in the future, 60 s leeway;
   - `nonce` equals the pending nonce;
   - `at_hash`, when present, matches the access token;
   - `sub` present and non-empty.
   Any failure: `id_token_invalid` (the specific check is logged at debug).
5. Userinfo when email, name or groups are missing and a `userinfo_endpoint` exists; its `sub` must
   equal the ID token's (`userinfo_sub_mismatch`).
6. Claims to `ExternalIdentity`: `email_verified` accepts a bool or the strings `"true"`/`"false"`;
   groups accept a list or a space or comma separated string; `picture` only when https; the claims
   snapshot drops tokens and is capped at 8 KB.
7. The generic resolver (section 3). Errors are translation keys in `strings.json`.

### 5. Frontend (`music-assistant/frontend`)

The frontend ships in lockstep with the server and never gates on `schema_version`
(`.github/instructions/music-assistant-frontend-standards.instructions.md:24`). New UI uses
shadcn-vue only (`README.md:55-67`). UI PRs carry screenshots (instructions file, line 19).

**Helpers.**

- `src/helpers/pkce.ts`: `createVerifier()` (43 to 128 chars from `crypto.getRandomValues`),
  `challengeS256(verifier)` via `crypto.subtle`, with a small pure-JS SHA-256 fallback because
  `crypto.subtle` is missing on insecure origins such as `http://192.168.1.10:8095`.
- `src/helpers/external_login.ts`: the pending record
  `{state, verifier, providerId, purpose, remoteId?, returnPath, createdAt}` in `sessionStorage`
  under `ma.external_login.<state>` with a `localStorage` mirror (TTL 10 minutes; the mirror covers
  an IdP that opens a new tab); `startExternalLogin()` refuses any `authorization_url` that is not
  `http:` or `https:` before navigating (today `Login.vue:1958` calls `location.replace` unchecked);
  `readExternalLoginReturn(location)` returns `{code, state}` or `{error, state}` only when a pending
  record with that state exists, always strips `code`, `state`, `error`, `error_description` with
  `history.replaceState`, and drops a `code` without a matching state.
- API wrappers in `src/plugins/api/index.ts` (`getSignInMethods`, `startAuthorization`,
  `exchangeAuthCode`, `getIdentities`, `unlinkIdentity`, `getAuthPolicy`, `updateAuthPolicy`) and
  the types in `interfaces.ts`. `AuthProviderType` in `interfaces.ts:1788-1791` is corrected (see
  the bug table).

**Login rebuilt in shadcn-vue.** `src/views/Login.vue` (2439 lines, Vuetify today) becomes a thin
host with the same three emits (`connected`, `authenticated`, `local-connect`, `Login.vue:484-496`)
and the exposed `handleAuthenticationError` (`Login.vue:1938-1940`), so `App.vue:8-14` stays
untouched. Parity PRs first: the connect logic (auto-connect order, remote id, guest and dashboard
codes) moves into `src/composables/useConnectFlow.ts` with the existing `tests/views/Login.test.ts`
suite kept green, then the template moves to shadcn. New pieces:

- `src/composables/useSignInMethods.ts`: loads `auth/signin_methods`, re-fetches on window focus,
  once 5 s after connect while the list has no redirect method, and on the event when signed in.
- `src/components/login/`: `LoginShell.vue`, `LoginProgress.vue`, `LoginServerSelect.vue`,
  `LoginSignIn.vue` (method buttons, separator, password form), `SignInMethodButton.vue`,
  `PasswordForm.vue` (TanStack form like `AccountStep.vue:151`, `:202`), `LoginRedirecting.vue`,
  `LoginError.vue`, `LoginGuestEnded.vue`.

Hand-back rules:

- The hand-back runs before any stored-token login. With a matching pending record it calls
  `auth/exchange` with the stored verifier, stores the token and emits `authenticated`.
- The rule that takes any `?code=` longer than 8 characters as a token (`Login.vue:1262-1274`) is
  deleted. The first-run setup keeps receiving its token in the response body
  (`useFirstRunSetup.ts:105-123`), and `/login` no longer hands the web app a JWT.
- `?local=1` always shows the password form and suppresses auto-launch.
- Auto-launch of `auto_launch_provider_id`: once per tab (a sessionStorage flag), never after logout,
  cancel or an error, never on ingress (`helpers/ingress.ts:9-15`), guest join or dashboard paths,
  and always with a visible "Use another way to sign in" escape.
- Remote mode: the same-tab flow. The pending record holds the remote id; on return the app
  reconnects by that id first and exchanges over the data channel.

**Profile (`src/views/UserProfile.vue`).** New `src/components/profile/LinkedIdentitiesSettings.vue`
(list, "Link" through a dropdown of unlinked redirect methods with purpose `link`, unlink with
confirmation, the last-method guard explained). `PasswordSettings.vue` asks the current password
for a self change (today it has only new and confirm fields) and becomes "Set a password" when
`has_password` is false. `PasskeySettings.vue` arrives with sub-issue 13.

**Admin (`src/views/settings/UserManagement.vue`, `src/components/users/`).** `CreateUserDialog.vue`
gains email and "How will they sign in?" (password, or no password with the email required and a
note that they sign in through a linked provider; invite links come later). `EditUserDialog.vue`
shows email and identities. The users table shows email and sign-in badges from `login_methods`.

**Settings > Sign-in methods.** Route `settings/sign-in` (name `signinsettings`) rendering
`Providers.vue` for type `auth` (`Providers.vue:236-240` and `AddProviderDialog.vue:175-179` map
labels per type today), plus `src/components/auth/AccountSignInRow.vue` (the "Music Assistant
account" row: password, later passkeys) and `src/components/auth/SignInPolicyCard.vue` (password
switch with the guard reason, auto-launch select, the `/?local=1` escape-hatch URL, and a warning
that the Apple TV app and older companion apps need a password). A section in
`src/helpers/settings_sections.ts` with scope `config.providers.write`; the policy card is read-only
without `users.manage`. Auth providers are excluded from the plugin section and from the add
dialog of other types.

**Tests (vitest).** `pkce` against the RFC 7636 appendix B vector; `external_login` store, strip and
guards; the portal callback functions (shared test vectors); the Login parity suite plus new cases
(hand-back, `?local=1`, auto-launch suppression, remote reconnect); settings page; linked
identities; admin dialogs; passkey UI gating (sub-issue 13).

### 6. Remote portal (`music-assistant/app.music-assistant.io`)

The portal is a Vite build with root `src` and `publicDir ../public` (`vite.config.ts:4-6`),
deployed to GitHub Pages on pushes touching `src/**` or `public/**` (`.github/workflows/deploy.yml:3-12`),
with the stable, beta and nightly frontend builds unpacked under `/<channel>/`
(`deploy.yml` build step) and the custom domain in `public/CNAME` (verified). A static file at
`public/auth/callback/index.html` is served at `https://app.music-assistant.io/auth/callback/`. GitHub
Pages serves the trailing-slash form; that exact URL is what IdPs register.

The page has no app bundle, a strict CSP in a meta tag (GitHub Pages sets no headers):
`default-src 'none'; script-src 'sha256-<inline>'; style-src 'unsafe-inline'`, and
`<meta name="referrer" content="no-referrer">`. Logic:

1. Read `code`, `state`, `error`, `error_description`. `state` must match `^[wn]\.[A-Za-z0-9_-]{20,128}$`,
   `code` `^[A-Za-z0-9._~-]{1,2048}$`, `error` `^[a-z_]{1,64}$`; `error_description` is shown as
   text only, cut to 300 characters. Anything else: "This sign-in link is not valid".
2. `n.` state: `location.replace("musicassistant://auth/callback?" + params)` plus an "Open the app"
   button with the same link.
3. A same-origin `window.opener`: `postMessage({type: "ma-external-login", code, state, error},
   location.origin)` and close.
4. Else `BroadcastChannel("ma-external-login")`: post, wait 400 ms for `{type: "ack", state}`;
   on ack show "You can close this tab".
5. Else `location.replace("/?" + params)`: the portal root (`src/main.ts`) forwards `code` and
   `state` to the saved channel build (`/<channel>/?remote_id=<saved>&code=...&state=...`, next to
   the existing `redirectToFrontend` calls at `main.ts:274` and `:389`), where the Login hand-back
   finishes the job.

The page never redirects to a URL taken from the query.

### 7. Mobile app, desktop app, Apple TV and python client

**Mobile (`music-assistant/mobile-app`, KMP).** Today the app does system-browser OAuth with
`musicassistant://auth/callback` (`auth/OAuthCallback.kt:32-35`, `:64-65`;
`androidApp/src/main/AndroidManifest.xml:40-47`; iOS `ASWebAuthenticationSession`, not ephemeral,
`iosApp/OAuthWebSession.swift:50-61`), takes the JWT from `?code=`
(`OAuthCallback.kt` `parse`), and renders only `builtin` and `homeassistant`
(`ui/compose/auth/AuthenticationPanel.kt:102-135`) (verified). Changes:

- `AuthMethod` and `AuthPolicy` models from `auth/signin_methods`, with a fallback mapping from
  `auth/providers` on servers below schema 85.
- Schema gate 85 for PKCE and `auth/exchange`; below it the current flow stays.
- `Pkce.kt` on cryptography-kotlin, already a dependency (`gradle/libs.versions.toml:49`,
  `:135-136`).
- The pending record (state, verifier, server id) persisted with a 10-minute TTL so a process death
  during the browser step survives.
- `OAuthCallbackResult.Code` carries `code` and `state`; then `auth/exchange`, then `auth {token}`.
- A generic "Sign in with {name}" button for every redirect method; the password tab is hidden when
  `password_enabled` is false; no auto-launch in v1.
- On the remote connection the IdP returns to the portal callback page, which forwards `n.` states
  to `musicassistant://auth/callback`.
- Separate item: iOS tokens move from `NSUserDefaults` (`iosMain/.../provideSettings.ios.kt:7-15`)
  to the Keychain.

**Desktop (`music-assistant/desktop-app`, Tauri).** The launcher page navigates the webview to the
server (`src-tauri/resources/index.html:498`); there is no deep-link plugin, only
`tauri-plugin-single-instance` (`src-tauri/Cargo.toml:48-49`) whose callback only focuses the window
(`src-tauri/src/lib.rs:472-480`) (verified). Changes:

- `tauri-plugin-deep-link` with scheme `musicassistant-desktop`, and the single-instance plugin's
  `deep-link` feature so a second launch hands the URL to the running instance.
- Commands `external_login_supported() -> bool`, `open_external_login(url)` (http and https only,
  opens the system browser through the opener plugin), `take_external_login_callback() -> Option<String>`
  (one-shot). The page is notified with the existing `window.eval` pattern (`lib.rs:248-250`), for
  example `window.__MA_EXTERNAL_LOGIN__ && window.__MA_EXTERNAL_LOGIN__()`.
- The frontend uses `musicassistant-desktop://auth/callback` as `return_url` when
  `external_login_supported()` is true, else today's in-webview flow.
- The server allowlist gains `musicassistant-desktop://` (`redirect_validation.py:18-26`).

**Apple TV.** Unchanged in v1: password and dashboard code. Sign in on another device is a later
sub-issue (16).

**Python client (`music-assistant/client`, at `21b5f55`).** Additive only: `generate_pkce_pair()`,
`exchange_code(state, code, code_verifier, device_name)`, `get_sign_in_methods()`,
`get_identities()`, `unlink_identity()`, `create_user(..., password=None, email=None)` next to
`Auth.create_user` (`auth.py:68`) and `get_user_providers` (`auth.py:104`); `login()` keeps
`POST /auth/login` (`auth_helpers.py:24`). Models bump. The Home Assistant integration needs nothing.

### 8. OHF sign-in relay and the `apple` and `google` auth providers (first follow-up)

The relay is a small OHF-hosted service (Cloudflare Worker with KV or equivalent, a new repo, for
example `music-assistant/signin-relay` at `https://signin.music-assistant.io`). Secrets: Apple Team
ID, Key ID, private key and Services ID; Google client id and secret. It acts as an OAuth
authorization server facade for MA installs.

| endpoint | caller | contract |
|---|---|---|
| `POST /register` | MA server | body `{provider: "apple" \| "google", state, nonce, code_challenge, return_origin}`; stored 10 minutes under `state`; `return_origin` is the only place the relay may send the browser |
| `GET /authorize?state=` | browser | looks the state up, redirects to Apple or Google with OHF's client, the MA nonce and `redirect_uri` = the relay callback; Apple with `response_mode=form_post` and scopes `name email` |
| `GET /callback/google`, `POST /callback/apple` | browser | exchanges the code upstream with OHF's secret (Apple: an ES256 client secret JWT), keeps the ID token and Apple's first-login `user` JSON under a one-time relay code, then sends the browser to `<return_origin>/auth/callback?code=<relay code>&state=<state>` |
| `POST /token` | MA server | body `{code, code_verifier}`; returns `{id_token, user?}` once when `S256(code_verifier)` equals the registered challenge |

Redirect rule at the relay: private IPs, loopback, `.local`, `.home.arpa` and
`app.music-assistant.io` origins redirect at once; other https origins first get a consent
interstitial ("You are signing in to Music Assistant at https://music.example.com"), the same rule
MA applies to external return URLs. Plain http is refused for public hosts. The relay rate-limits
`/register` per IP and stores nothing beyond the 10-minute entries.

MA side (`providers/apple/`, `providers/google/`, sharing `helpers/signin_relay.py`):
`build_authorization_url` registers the state, nonce and the relay challenge (the relay verifier is
kept in `PendingLogin.provider_data`) and returns `<relay>/authorize?state=`. `return_origin` is the
callback base of section 2. `complete_authorization` calls `/token` and verifies the ID token with
PyJWT against the upstream JWKS (Apple `https://appleid.apple.com/auth/keys`, Google
`https://www.googleapis.com/oauth2/v3/certs`, cached like the OIDC client):

- signature with `RS256`;
- `iss` = `https://appleid.apple.com`, or `https://accounts.google.com` / `accounts.google.com`;
- `aud` = OHF's client id (Apple Services ID, Google client id), fixed in each provider's
  `constants.py`;
- `nonce` = the pending login's nonce, so a token minted for one install cannot be replayed at
  another;
- `exp` and `iat` with 60 s leeway; `sub` present.

Apple delivers `email_verified` and `is_private_email` as strings and the name only on the first
login (from the `user` JSON). Google's `email_verified` is a real bool and an `hd` claim is
available. Both providers have no config beyond adding them, set `supports_remote_app`, need
internet, and are unavailable with `relay_unreachable` when the relay cannot be reached. Google's
"own client" path stays an OIDC preset; since Google's `sub` is the same for every client and the
issuer is the same, a later switch keeps existing links.

OHF tasks in the sub-issue: hosting and secrets custody, Apple Services ID with the relay domain,
Google OAuth brand verification with a privacy policy and only the basic scopes.

### 9. Native passkeys (first follow-up)

Dependency `webauthn` (py_webauthn 3.0.1, BSD-3, synchronous and CPU-light; dependencies pyasn1,
cbor2, cryptography, pyOpenSSL; from the PyPI metadata of 3.0.1; not a server dependency today). Schema
`auth.db` v7 adds:

```sql
CREATE TABLE IF NOT EXISTS webauthn_credentials (
    credential_id TEXT PRIMARY KEY,      -- base64url
    user_id TEXT NOT NULL,
    rp_id TEXT NOT NULL,
    public_key BLOB NOT NULL,
    sign_count INTEGER NOT NULL DEFAULT 0,
    transports json NOT NULL DEFAULT '[]',
    aaguid TEXT,
    name TEXT,
    backup_eligible INTEGER NOT NULL DEFAULT 0,
    backup_state INTEGER NOT NULL DEFAULT 0,
    created_at TEXT NOT NULL,
    last_used_at TEXT
);
CREATE INDEX IF NOT EXISTS idx_webauthn_user ON webauthn_credentials(user_id);
```

Added to the delete and prune lists. Same migration rule as v6.

RP ID rules: accepted RP IDs are the hostname of `external_url` when it is a domain (not an IP) and
`app.music-assistant.io` for the remote app. The RP ID is chosen from the transport (REMOTE: the app
domain; DIRECT: the external domain when the request origin's host equals it) and stored per
credential; the expected origin is the matching `https://` origin. No usable RP ID: the method is
unavailable with `passkey_requires_public_url`. On the remote app every remote user shares that RP
ID, so a passkey there is the user's explicit choice per account, and `user.name` is
`<username> @ <server name>` so the authenticator lists them apart.

Commands (challenges in the pending store, 300 s, single use):

```text
auth/passkey/register/options(name: str | None = None)  -> {"challenge_id": str, "options": dict}   # authenticated
auth/passkey/register/verify(challenge_id: str, credential: dict) -> PasskeyInfo        # authenticated
auth/passkey/login/options() -> {"challenge_id": str, "options": dict}                  # unauthenticated, discoverable, userVerification=required
auth/passkey/login/verify(challenge_id: str, credential: dict, device_name: str | None = None)
    -> {"success": true, "access_token": str, "user": {...}}
auth/passkey/list(user_id: str | None = None) -> list[PasskeyInfo]
auth/passkey/rename(credential_id: str, name: str) -> None
auth/passkey/delete(credential_id: str, user_id: str | None = None) -> None
```

Login verify checks the sign counter (a regression is logged, not fatal for synced passkeys with
counter 0), the user is enabled, and the rate limiter. Passkeys count as a way in for the
last-method guard and as an admin identity for the policy guard.

Frontend: "Sign in with a passkey" only when `window.isSecureContext`, `PublicKeyCredential`
exists and the hostname matches an accepted RP ID; conditional UI (autofill) where supported; a
`PasskeySettings.vue` profile card with add, rename and delete. In the mobile app the ceremony runs
in the system browser session against the server origin; native passkeys need `webcredentials:`
associated domains and come later.

### 10. Later sub-issues (specified, not built)

**IdP session re-validation (15).** Per-provider option `revalidate_sessions`; requests
`offline_access`; the refresh token is Fernet-encrypted into `user_identities.refresh_token`;
`auth_tokens.identity_id` added in a DB bump. During the existing hourly sliding renewal
(`_refresh_token_expiration`, `auth.py:2522`) the server refreshes at the IdP; `invalid_grant`
revokes that identity's tokens; transport errors fail open for 7 days. Back-channel logout
afterwards.

**Sign in on another device (16).** MA acts as an RFC 8628-style device authorization server for
its own clients: `auth/device/start` (device code, 8-character user code, `verification_uri` and
`verification_uri_complete` for a QR code), `auth/device/poll`, `auth/device/approve`,
`auth/device/deny`; web route `#/device`. The Apple TV app shows the QR code.

**Invite links (17).** A one-time link or code valid 7 days that pre-binds a password-less account
to the first identity signing in with it. Removes the email requirement of decision 8.

**Future identity provider on the home OS.** A Supervisor discovery payload `{issuer}` auto-creates
an `oidc` instance with preset `home_os` and registers MA as a client through RFC 7591 dynamic
registration. Identities are keyed by issuer and subject, so links survive. The same IdP serves
third-party apps such as Immich. Nothing in this epic blocks it.

## Phasing

MVP: sub-issues 1 to 12. First follow-ups right after the MVP: 13 and 14. Later: 15 to 17.

1. Models: `ProviderType.AUTH`, the auth dataclasses, `User.email`, `has_password`,
   `login_methods`, `AuthProviderType` additions with `_missing_` to UNKNOWN; release and server pin
   bump.
2. Server: the `auth` provider type and the sign-in flow foundation: `AuthProvider` base class,
   `login_flow.py` stores and PKCE, callback base per transport, the fixed callback URL, `hass_auth`
   replacing the in-core Home Assistant login, one-time code, `auth/exchange` and
   `POST /auth/token`, remote completion, `/login` and `/setup` compatibility, state prefixes, error
   hand-back. Fixes support#6601 and the backlog#153 class for sign-in.
3. Server: accounts, identities and policy: DB v6, the generic resolver and linking rules, accounts
   without a password (email required), identity commands, current password on a self change,
   policy with guard, fail-open and admin break-glass, `auth/signin_methods`, schema 85.
4. Frontend: new login page: parity PRs (logic into `useConnectFlow`, template in shadcn-vue), then
   helpers, sign-in method buttons, PKCE hand-back, `?local=1` and auto-launch.
5. Remote portal: the callback page in app.music-assistant.io, portal forwarding, frontend remote
   completion.
6. Server: `oidc` auth provider (client, setup flow with presets and the redirect URI listing, tests
   against a fake IdP on a pytest-aiohttp server), then the test sign-in and link step.
7. Frontend: Settings > Sign-in methods (type `auth` section, account row, policy card) and
   exclusion from the plugin lists.
8. Frontend: linked identities in the profile, set or change password, admin create without a
   password, email, identities in the edit dialog, sign-in badges in the users table.
9. Mobile app: methods, PKCE, exchange, schema gate, `n.` hand-back.
10. Desktop app: system browser and `musicassistant-desktop://` deep link; frontend bridge.
11. Python client: additive helpers and models bump.
12. Docs (music-assistant.io): sign-in methods, adding an IdP (per preset, including
    `email_verified` behaviour and the redirect URIs), the remote app, the policy and the escape
    hatch.
13. Passkeys: server ceremonies and credential table, frontend sign-in button and profile card,
    docs.
14. OHF sign-in relay (infra, new repo, secrets, Google brand verification) and the `apple` and
    `google` auth providers in the server, docs.
15. IdP session re-validation.
16. Sign in on another device (Apple TV).
17. Invite links.

A separate research issue outside the epic looks at one provider carrying several types, steered by
features, as a possible future provider model. The `auth` type is designed so it can be absorbed by
it.

### Prerequisite neutral bug-fix PRs (outside the epic, now)

| bug | where (verified) | fix |
|---|---|---|
| `auth/user/providers` returns the builtin row, whose `provider_user_id` is the PBKDF2 password hash | `controllers/webserver/auth.py:1625-1639`; hash stored at `helpers/auth_providers.py:476-477` | leave builtin rows out of the response (fixed by [server#6649](https://github.com/music-assistant/server/pull/6649), which returns the builtin row with an empty `provider_user_id` instead) |
| `auth/tokens` returns `token_hash` | `auth.py:823-829`; field in models `auth.py:138` | leave the hash out of the response (fixed by [server#6649](https://github.com/music-assistant/server/pull/6649), which returns an empty `token_hash` instead) |
| `PATCH /auth/me` skips the system-user guard and the username checks of `auth/user/update` | `controller.py:1127-1160` versus `auth.py:1550-1551` | route through the same rules, or drop the endpoint (fixed by [server#6671](https://github.com/music-assistant/server/pull/6671), which drops the endpoint and gives `auth/user/update` the username rules it lacked) |
| `POST /auth/logout` deletes the token row but leaves its websockets connected | `controller.py:1101-1117` versus `auth.py:1615-1621` | call `disconnect_websockets_for_token` (fixed by [server#6650](https://github.com/music-assistant/server/pull/6650)) |
| Home Assistant login of a disabled user: the username path builds the user from the raw row without the enabled check, so a token is minted or `update_user`'s assert fails | `auth_providers.py:817-845`, `auth.py:600` | refuse disabled users with a clear error (fixed by [server#6651](https://github.com/music-assistant/server/pull/6651)) |
| Ingress user resolution exists twice | `auth_middleware.py:120-175`, `websocket_client.py:521-567` | one shared helper (fixed by [server#6651](https://github.com/music-assistant/server/pull/6651); [server#6657](https://github.com/music-assistant/server/pull/6657) also shares the Home Assistant user mapping between Ingress and the Home Assistant login) |
| Websocket `auth/authorization_url` does not validate `return_url` | `auth.py:1066-1091` versus `controller.py:1184-1188` | validate like the HTTP route (fixed by [server#6650](https://github.com/music-assistant/server/pull/6650)) |
| Webserver README is out of date: bcrypt instead of PBKDF2, 10-year long-lived tokens, opaque tokens instead of JWTs, a remote OAuth polling flow that does not exist, the provider-id callback | `controllers/webserver/README.md` (lines 59, 64, 68, 185-203, 421-423) | rewrite now (documentation, not a code bug); sub-issue 2 updates it again |
| Frontend `AuthProviderType.OAUTH_HOMEASSISTANT = "oauth_homeassistant"` while the server sends `homeassistant` | frontend `src/plugins/api/interfaces.ts:1788-1791`, models `auth.py:31-32` | correct the value |

## Risks and open points

- **OHF infra decision for the relay.** Hosting, custody of the Apple key, Google brand
  verification and a privacy policy. Sub-issue 14 cannot start before it. Nothing in the MVP
  depends on it.
- **Shared remote origin.** Every remote user shares `app.music-assistant.io`. The callback page
  must never redirect to a URL from the query, must validate state and code shapes, and must not
  load third-party script. Remote passkeys share one RP ID; a credential only verifies at the server
  that stored its public key, and the user name carries the server name.
- **Hairpin NAT and registered URIs.** Preferring a known request origin covers an unreachable
  `external_url` on the LAN, but every callback URL a household uses must be registered at its IdP.
  The setup flow lists all three.
- **`email_verified` semantics differ per IdP.** Keycloak admins must tick "email verified";
  Authentik, Authelia and Pocket ID mark emails verified. Documented per preset. Only an email an
  admin set ever auto-links.
- **Startup gap.** Providers load after the webserver (`mass.py:337`, `:371-376`), so redirect
  methods appear a few seconds after boot. The plan said the login page refreshes on the
  sign-in-methods event, but unauthenticated connections receive no events
  (`websocket_client.py:173-177`). The login page re-fetches on focus and once after 5 s instead;
  the event serves the settings page.
- **Password switched off.** The Apple TV app and older companion apps only know passwords;
  non-admin users on them lose access. The policy card says so; admins keep break-glass and
  `/?local=1` always shows the form.
- **Legacy JWT-in-URL path.** Kept for clients without a challenge (external clients, old mobile
  apps, `login.html` with a return URL). Measured through the debug log before a deprecation.
- **Home Assistant link by username** stays (decision 6) for `hass_auth` only. A Home Assistant user
  named like an MA user still gets that account. Documented.
- **iOS standalone PWA.** The hand-off to Safari and back needs a device test with Home Assistant,
  Authentik and Google before sub-issue 4 closes.
- **`Provider.available` name clash** handled by `sign_in_available` on the base class.
- **Safe mode** has no Home Assistant button because `hass` is not loaded there; password sign-in
  remains.
- **Old peers and `AuthProviderType`.** Old models map `oidc` to `builtin`. New identity types only
  appear in new commands (`auth/user/identities`, `auth/signin_methods`); `auth/providers` lists new
  methods, which the old mobile app ignores because it matches the type strings
  (`AuthenticationPanel.kt:102-135`).
- **Open interpretation.** The plan's last-method guard ("refused when it would leave a non-admin
  with no way in") is read here as: a user cannot remove its own last way in; an admin with
  `users.manage` may, and is told the account has none left. To confirm in sub-issue 3.

## Verification

- Server: `tests/controllers/webserver/test_login_flow.py` (store TTL, single use and eviction;
  S256; the callback base matrix per transport and origin; Home Assistant `client_id` per target;
  `hass_auth` following `hass` load and unload; the legacy path byte-identical to today; code path
  failures; remote completion; rate limiting), next to the existing `test_auth_callback.py` and
  `test_ingress_auth.py`. Migration from a v5 fixture run twice. The linking matrix against a fake
  auth provider. Policy guard and fail-open. Passkey ceremonies with py_webauthn test vectors.
  `tests/providers/oidc/` with a fake IdP on a pytest-aiohttp test server: every ID token negative
  case of section 4, `kid` rotation, RFC 9207, userinfo, the three client auth methods, unload
  dropping pending logins. The relay client against a fake relay. Full suite
  `pytest -n auto --dist loadfile` and `pre-commit run --all-files`.
- End to end: Pocket ID or Authentik in Docker against a scratch MA instance on the direct LAN
  address, on a configured external URL and on the remote app through the portal callback; Home
  Assistant login through the remote app; a passkey on the remote app; Apple and Google through the
  relay's staging; the mobile app through the `n.` hand-back; the desktop deep link on macOS,
  Windows and Linux.
- Frontend: the vitest suites of section 5; screenshots in every UI PR.
