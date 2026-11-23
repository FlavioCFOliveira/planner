---
title: Security
status: draft
last-reviewed: 2026-11-23
related: [architecture.md, database-schema.md, data-integrity-and-concurrency.md, web-api.md, cli.md, operations.md, ../functional-spec/users-and-identity.md, ../functional-spec/cli-authentication.md, ../functional-spec/actors-and-roles.md]
---

# Security

## Overview

Trust boundaries: the browser and the CLI reach the server only through the TLS-terminating reverse proxy; the server trusts forwarded headers only from configured proxies; the Identity Provider is trusted for identity only (BR-IDN-001). Source ids `[Sn]` refer to the [source register](database-schema.md#sources). Decider key: [README.md](README.md#conventions).

## Decisions

| # | Decision | Rationale | By |
|---|----------|-----------|----|
| SE1 | WEBSITE login: OIDC authorization code flow with PKCE (S256), against any standards-compliant provider. Endpoints come from discovery; when the provider offers none, the authorization, token, UserInfo and JWKS endpoints MAY be configured explicitly. Issuer, client id and client secret are server configuration. Library: `openidconnect` 4.1.0 [S52]. Keycloak 26.8.2 [S46] is the reference provider for tests. | FR-IDN-001, FR-IDN-002, FR-IDN-024. Discovery is optional for providers [S36 §4], so explicit endpoints keep FR-IDN-024 satisfiable. PKCE needs no provider-side detection: a client sends the parameters to every server, and a server that does not support PKCE falls back to plain OAuth 2.0 [S37 §5]. | U |
| SE2 | Server-side sessions with an opaque cookie; idle timeout 8 h, absolute timeout 7 days. | [Below](#website-sessions). | U |
| SE3 | CSRF defence: `SameSite=Lax`, per-session synchronizer token, `Origin` check. | [Below](#csrf). | U |
| SE4 | CLI token secret `plnr_` + base64url without padding of 32 CSPRNG bytes; stored as SHA-256; sent as `Authorization: Bearer` with `X-Device-Id`, over HTTPS only. | [Below](#cli-tokens). | U |
| SE5 | In-memory rate limiting per token and per client IP with `governor` 0.10.4 keyed limiters in the server's own axum middleware; `429` with `Retry-After`. | [Below](#rate-limiting). Single instance (A1). | U (values: B; middleware: B) |
| SE6 | TLS terminates at a reverse proxy. The server listens on plain HTTP on an internal interface and trusts `X-Forwarded-For`/`X-Forwarded-Proto` only from configured proxies. | [Below](#reverse-proxy-and-client-address). | U |

## OIDC login

Endpoints are WEBSITE routes, not `/api/v1`.

1. **Provider metadata.** If `PLANNER_OIDC_AUTH_ENDPOINT` is set, the server uses the explicitly configured endpoints and does not attempt discovery (all of authorization, token and JWKS are then required; UserInfo is optional). Otherwise it reads `<issuer>/.well-known/openid-configuration` [S36 §4] on first use and caches it for the process lifetime; the document's `issuer` MUST equal the configured issuer exactly [S36 §4.3]. The JWKS is fetched from the JWKS URI and re-fetched when an ID Token names an unknown key. With explicit endpoints the client is built with `Client::new` and the endpoint setters (`set_auth_uri`, `set_token_uri`, `set_user_info_url`) [S52]. (Selection rule: B.)
   HTTP to the provider uses `reqwest` 0.13.5 with `rustls` [S53], passed to `openidconnect` as an async HTTP client closure, with `openidconnect`'s default features disabled: its `reqwest` feature targets reqwest 0.12 through `oauth2` 5.0.0 [S52]. Redirects are not followed, as `openidconnect` requires against SSRF [S52].
2. **`GET /auth/login`.** Generate `state` and `nonce` (each 32 CSPRNG bytes, base64url) and a PKCE `code_verifier` (32 CSPRNG bytes, base64url: 43 characters, within 43–128 [S37 §4.1]). Store them in the [pre-login cookie](#pre-login-cookie). Redirect to `authorization_endpoint` with `response_type=code`, `scope=openid profile email`, `client_id`, `redirect_uri` = `PLANNER_PUBLIC_URL` + `/auth/callback`, `state`, `nonce`, `code_challenge` = BASE64URL(SHA256(verifier)), `code_challenge_method=S256` [S37 §4.2]. (`nonce` is OPTIONAL in the code flow [S21 §3.1.2.1]; planner always sends it.)
3. **`GET /auth/callback`.** Refuse unless the pre-login cookie is present, decrypts, has not expired, and its `state` equals the `state` parameter. The cookie is then expired in the response, so it is used once. Exchange the code at `token_endpoint` with the `code_verifier`. Client authentication: `client_secret_basic` when `token_endpoint_auth_methods_supported` is omitted or lists it (it is the default [S21 §9, S36 §3]), otherwise `client_secret_post` when listed; if neither is available the login fails. (B)
4. **ID Token validation** per OIDC Core §3.1.3.7 [S21]: signature against the JWKS; `iss` equals the configured issuer; `aud` contains the client id; current time before `exp`; `nonce` equals the stored value.
5. **Profile.** Look up `users` by `sub` (`idp_subject`).
   - Deactivated: refuse the login; create no session (FR-IDN-014).
   - Absent: create the profile (FR-IDN-017) in one transaction with its audit entry `create` `user` (attributed to the new user, `deviceid` `GUI`). `name` and `email` come from the UserInfo endpoint, where the `profile` and `email` scope claims are returned when an access token is issued [S21 §5.4]; when the provider advertises no `userinfo_endpoint` or the call fails, the ID Token claims are used. Values are truncated to 64 code points; an absent value stays NULL (FR-IDN-004, FR-IDN-023). A UserInfo failure never blocks the login (FR-IDN-023). The IdP is not read again afterwards (FR-IDN-005).
6. **Session.** Create a new session ([below](#website-sessions)); a new id is issued at every login. Redirect to the WEBSITE home.

### Pre-login cookie

Carries `state`, `nonce`, the PKCE verifier and an expiry from `/auth/login` to `/auth/callback`. (U unless marked.)

| Aspect | Rule |
|--------|------|
| Protection | `axum-extra` 0.12.6 `PrivateCookieJar` (features `cookie` and `cookie-private`), which encrypts every cookie with a `Key` using the `cookie` crate's private jar: authenticated encryption assuring confidentiality, integrity and authenticity [S55]. A cookie that fails decryption is refused. (U) |
| Lifetime | 10 minutes: the expiry inside the payload is checked, and `Max-Age=600`. |
| Attributes | Name `__Host-planner_login` (B); `HttpOnly; Secure; SameSite=Lax; Path=/`. `Lax` cookies are sent on the provider's top-level `GET` redirect back to the callback [S43]. |
| Key | `PLANNER_COOKIE_KEY` (secret): standard base64 [S40 §4] of at least 64 bytes, the minimum `cookie::Key::from` requires (it panics on less; `Key::try_from` returns an error) [S55]. The server loads it with `Key::try_from` and refuses to start on error. Optional `PLANNER_COOKIE_KEY_PREVIOUS`, same format, is used for decryption only: a cookie that the current key cannot decrypt is tried with the previous key. (U; base64: B) |
| Rotation | Move the current key to `PLANNER_COOKIE_KEY_PREVIOUS`, set a new `PLANNER_COOKIE_KEY`, restart; remove the previous key after at least 10 minutes. Without a previous key, logins in progress during the restart fail and the user starts again. (B) |

### First administrator

FR-ACC-012, FR-ACC-017. At every successful login of the `sub` equal to `PLANNER_FIRST_ADMIN_SUB`, in the login transaction: if no `users` row has `is_admin = 1` (locking read, [data-integrity-and-concurrency.md](data-integrity-and-concurrency.md#rule-enforcement)), the user becomes an administrator. If the profile is created in that login, it is inserted with `is_admin = 1` and audited as `create` `user` and `modify` `user`; if it exists, `is_admin` is set to 1 and audited as `modify` `user`. Entries are attributed to that user, `deviceid` `GUI`, one `createddt`. A deactivated user is refused before this rule (FR-IDN-014). Once any administrator exists the rule never fires, so the configured user receives no further special treatment (FR-ACC-017). (U)

## WEBSITE sessions

Table: [database-schema.md](database-schema.md#central-database-planner) `sessions`. Domain type: [domain-model.md](domain-model.md#session).

| Aspect | Rule | By |
|--------|------|----|
| Session id | 32 CSPRNG bytes, base64url without padding, as the cookie value. Only SHA-256(id) is stored (`id_hash`). | U (encoding: B) |
| Cookie | Name `__Host-planner_session`. Attributes `HttpOnly; Secure; SameSite=Lax; Path=/`, no `Domain`, `Max-Age` = remaining absolute lifetime. The `__Host-` prefix makes browsers reject the cookie unless it is `Secure`, host-only and `Path=/` [S43]. `Secure` is always set: browsers reach planner only over HTTPS (SE6). | Attributes: U. Name, prefix, `Max-Age`: B |
| Idle timeout | 8 h since `lastseendt`. | U |
| Absolute timeout | 7 days since `createddt`. | U |
| Per request | Look up by `id_hash`. If either timeout has passed, delete the row and redirect to login. Otherwise set `lastseendt` to now, in its own statement before the request's transaction (L3 places `sessions` last; this keeps it out of the operation's locks). | B |
| Logout | `POST /auth/logout` (CSRF-protected) deletes the row and expires the cookie. | U |
| Deactivation | Deletes all the user's sessions in the deactivation transaction ([data-integrity-and-concurrency.md](data-integrity-and-concurrency.md#other-composite-operations)), so FR-IDN-014 holds immediately. | U |
| Expired rows never presented again | Deleted by the hourly [session purge](architecture.md#periodic-tasks). | U |

`SameSite` is defined in draft-ietf-httpbis-rfc6265bis, not yet an RFC [S43].

## CSRF

Applies to every WEBSITE request with method `POST`, `PUT`, `PATCH` or `DELETE`. WEBSITE `GET` requests never change state. `/api/v1` ignores cookies entirely, so it needs no CSRF defence.

1. `SameSite=Lax` on the session cookie (above).
2. Synchronizer token: the request MUST carry the session's `csrf_token` (base64url) in the form field `csrf_token` or the header `X-CSRF-Token` (htmx sends it through `hx-headers`). It is compared in constant time. (Field and header names: B.)
3. `Origin`: the request MUST carry an `Origin` header equal to the origin of `PLANNER_PUBLIC_URL`. A missing `Origin` is refused. (Refusal of a missing header: B.)

A failure returns 403 with code `CSRF_REJECTED`.

## CLI tokens

Requirements: FR-AUTH-001…026, NFR-SEC-001.

| Aspect | Rule |
|--------|------|
| Secret | `plnr_` followed by base64url without padding [S40 §3.2, §5] of 32 CSPRNG bytes: 48 ASCII characters matching `^plnr_[A-Za-z0-9_-]{43}$`. Shown once at issue on the WEBSITE (FR-AUTH-017), never stored. The prefix makes leaked secrets recognisable by scanners and in logs. |
| Storage | `tokens.secret_hash` = SHA-256 of the full secret ([database-schema.md](database-schema.md#central-database-planner)). |
| Transport | `Authorization: Bearer <secret>` [S41 §2.1]; the secret's characters are all valid `b64token` characters. `X-Device-Id: <24 characters>` on every request (FR-AUTH-014). HTTPS only [S41 §5.3], except on the loopback interface: the CLI accepts `http` only for `127.0.0.1`, `::1` and `localhost` ([cli.md](cli.md#configuration-and-credentials)) (U). The server refuses a token request with 403 `HTTPS_REQUIRED` unless its trusted proxy received it over HTTPS (`X-Forwarded-Proto` = `https`) or it is a direct loopback request: TCP peer on a loopback address and no `X-Forwarded-For` header, which a proxy always appends (B; loopback exception: B). |
| Validation | A malformed `Authorization` header is refused without a database read. Otherwise: look up by hash; refuse when unknown, revoked, expired (`expiresdt` ≤ now, FR-AUTH-026), or when `X-Device-Id` differs from `tokens.deviceid` (FR-AUTH-010, FR-AUTH-011). Refusal: 401 with `WWW-Authenticate: Bearer error="invalid_token"` [S41 §3.1]. Writes repeat the check under locks ([data-integrity-and-concurrency.md](data-integrity-and-concurrency.md#permission-check-in-writes)). |

## Authorisation

Effective permission in a project, evaluated per request (FR-ACC-002…006, FR-AUTH-008, FR-AUTH-018, FR-AUTH-022, BR-ACC-003):

| Caller | Administrator | Role in project | Effective level |
|--------|---------------|-----------------|-----------------|
| Session | yes | any or none | every operation |
| Session | no | `owner` / `member` / `viewer` | the role's rights ([actors-and-roles.md](../functional-spec/actors-and-roles.md#permission-summary)) |
| Session | no | none | none |
| Token with grant *g* | yes | any or none | *g* (FR-AUTH-022) |
| Token with grant *g* | no | `owner` / `member` | *g*, further limited by the role's rights |
| Token with grant *g* | no | `viewer` | `read` |
| Token without grant, or no role (non-admin) | — | — | none |

An operation needs `read` or `read+write` ([web-api.md](web-api.md#resource-map)) and, where the functional specification says so, a role (for example `owner` for settings, FR-KBN-016). Operations excluded by FR-CLI-006 are not part of `/api/v1`, so no token reaches them (FR-AUTH-019, FR-AUTH-024). Two operations are not project-scoped in the usual way:

- User search (FR-ACC-019, FR-ACC-020): `read` suffices, because it changes no state (FR-AUTH-018); the caller needs the `owner` role in the project, or administrator status (BR-ACC-003). (Level: U.)
- A user's photo and name to display (FR-AUTH-027: `displayname`, else `name`, as FR-IDN-021): any valid token, whatever its grants; read-only.

## Rate limiting

Token buckets held in server memory (single instance, A1), implemented with `governor` keyed rate limiters (`RateLimiter::keyed`, `check_key`) [S50] in the server's own axum middleware. `tower_governor` 0.8.0 supports axum 0.8 [S51] but is not used: its `SmartIpKeyExtractor` reads `X-Forwarded-For` without the trusted-proxy rule of [below](#reverse-proxy-and-client-address), and the failed-authentication bucket counts responses, not requests (B). A request passes every bucket that applies to it; otherwise 429 with code `RATE_LIMITED` and `Retry-After` in whole seconds [S42 §4]. Values are configuration ([operations.md](operations.md#configuration)). Defaults: B.

| Bucket | Key | Applies to | Default rate | Default burst | Rationale |
|--------|-----|------------|--------------|---------------|-----------|
| General | client IP | every request, before authentication | 200/s | 400 | Equal to NFR-PERF-002, so one host or NAT running many agents can reach the required load. |
| Token | token id | authenticated `/api/v1` requests | 20/s | 40 | 10× the mean share of an active user (200 req/s over ≥ 100 active users, NFR-PERF-001/002), leaving room for agent bursts. |
| Login | client IP | `/auth/login`, `/auth/callback` | 10/min | 10 | Stricter on authentication endpoints. |
| Failed token authentication | client IP | requests refused with 401 | 10/min | 10 | Bounds hash lookups from invalid tokens. Guessing a 256-bit secret is infeasible regardless. |

Idle keys are evicted by `retain_recent` [S50], called by the hourly task, which bounds memory. (B; call site: B)

## Reverse proxy and client address

| Aspect | Rule |
|--------|------|
| Listener | Plain HTTP on `PLANNER_LISTEN_ADDR`, an internal interface. |
| Trusted proxies | `PLANNER_TRUSTED_PROXIES`: list of CIDR ranges. |
| Client IP | If the TCP peer is trusted: walk `X-Forwarded-For` from right to left, skipping trusted addresses; the first untrusted address is the client IP; if none, the peer. If the peer is not trusted: `X-Forwarded-For` and `X-Forwarded-Proto` are ignored and the client IP is the peer. Used for rate limiting and logs. (B) |
| Scheme | The last `X-Forwarded-Proto` value from a trusted peer; `http` otherwise. Used for the HTTPS check on tokens. |
| Public origin | `PLANNER_PUBLIC_URL` (an `https` URL) is the only source for the OIDC `redirect_uri` and the `Origin` check. The `Host` header is not used. (B) |
| Proxy obligations | Terminate TLS; append the peer to `X-Forwarded-For`; set `X-Forwarded-Proto`; not expose the internal listener. ([operations.md](operations.md#deployment)) |

`X-Forwarded-For` and `X-Forwarded-Proto` are de facto headers; no governing standard is cited for them.

Secrets in logs: [operations.md](operations.md#observability).
