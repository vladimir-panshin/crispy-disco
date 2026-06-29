# Warden

**Session-based authentication service in Go** — TOTP 2FA, single-use recovery codes, device-aware session management, and brute-force protection. Built on PostgreSQL and Redis.

![Go](https://img.shields.io/badge/Go-1.26-00ADD8?logo=go&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

Warden is a focused authentication backend: it does identity and access — registration, login, two-factor authentication, and session lifecycle — and deliberately nothing else. No user profiles, no roles, no feature creep. The goal is a clean, production-minded core that's easy to read and easy to build on.

---

## Features

- **Email + password authentication** with bcrypt password hashing.
- **Session-based auth backed by Redis** — sessions are first-class and revocable (unlike stateless JWTs), which makes "log out", "log out everywhere", and per-device session listing possible.
- **TOTP two-factor authentication** — QR-code provisioning for authenticator apps, plus single-use base32 recovery codes for account recovery.
- **Race-free recovery codes** — a recovery code is consumed in a single atomic SQL statement, so it can't be redeemed twice under concurrent requests.
- **Two-stage 2FA login** — a dedicated intermediate session state that cannot reach protected routes until the second factor is verified.
- **Brute-force protection** — per-IP rate limiting on all authentication endpoints.
- **Reverse-proxy aware** — configurable trusted proxies for correct client-IP resolution behind Cloudflare/nginx.
- **Runs as a non-root container**, ships as a tiny static binary on Alpine.

---

## Tech stack

| Concern | Choice |
|---|---|
| Language | Go |
| HTTP | Gin |
| Database | PostgreSQL (via `pgx`) |
| Sessions & rate limiting | Redis (via `go-redis`, `ulule/limiter`) |
| 2FA | TOTP (`pquerna/otp`) |
| Password hashing | bcrypt (`golang.org/x/crypto`) |

---

## Architecture

A small, layered codebase:

```
cmd/server      entry point, wiring, config
auth            registration, login, 2FA verification (HTTP layer)
account         account management: password, email, sessions, 2FA setup
session         Redis-backed session manager (create, list, revoke)
middleware      auth guard, pending-2FA guard, rate limiter
db              PostgreSQL access (accounts, recovery codes)
models          request DTOs and validation rules
```

Handlers stay thin — they parse HTTP, call domain logic, and shape responses. Cryptographic logic (TOTP, recovery-code generation) lives apart from the HTTP layer so it can be reasoned about and tested in isolation.

### Two-stage 2FA login flow

```
POST /auth/login  (email + password)
   ├─ 2FA disabled → full session cookie ........................ done
   └─ 2FA enabled  → pending-2FA session cookie + {requires_2fa:true}
                        │
                        ▼
         POST /auth/2fa            POST /auth/2fa/recovery
         (TOTP code)        or     (single-use recovery code)
                        │
                        ▼
         pending session is replaced with a full session
```

The pending session is a distinct state — it authenticates *who is mid-login* but grants access to nothing except the verification step. Protected routes reject it.

---

## Getting started

### With Docker (recommended)

```bash
cp .env.example .env
docker compose up --build
```

This starts PostgreSQL, Redis, and the service. The schema is applied automatically on first run. The API is then available at `http://localhost:8080`.

### Locally

Requires Go 1.26+, a running PostgreSQL, and Redis.

```bash
cp .env.example .env          # then edit values for your local setup
psql "$DATABASE_URL" -f schema.sql
go run ./cmd/server
```

---

## Configuration

All configuration is via environment variables.

| Variable | Description |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string |
| `REDIS_URL` | Redis address (`host:port`) |
| `SERVER_PORT` | HTTP port (default `8080`) |
| `TRUSTED_PROXIES` | Comma-separated CIDRs of trusted reverse proxies. Empty = trust direct connections only. |

> **Behind a reverse proxy?** Rate limiting keys on the client IP. If Warden runs behind Cloudflare/nginx, set `TRUSTED_PROXIES` to the proxy's ranges (for Cloudflare, see <https://www.cloudflare.com/ips/>) — otherwise the limiter sees the proxy's IP for every request, or, if misconfigured to trust everyone, can be bypassed with a spoofed `X-Forwarded-For` header.

---

## API

Base path: `/api/v1`. Sessions are carried in an `HttpOnly` cookie (`session_id`); a `Bearer` token is also accepted.

### Authentication (public)

| Method | Path | Description |
|---|---|---|
| `POST` | `/auth/register` | Create an account, start a session |
| `POST` | `/auth/login` | Log in; returns `{requires_2fa:true}` if 2FA is enabled |
| `POST` | `/auth/2fa` | Complete login with a TOTP code |
| `POST` | `/auth/2fa/recovery` | Complete login with a recovery code |
| `POST` | `/auth/logout` | End the current session |

### Account (authenticated)

| Method | Path | Description |
|---|---|---|
| `GET` | `/account/me` | Current account |
| `PATCH` | `/account/me/password` | Change password |
| `PATCH` | `/account/me/email` | Change email |
| `DELETE` | `/account/me` | Delete account |
| `POST` | `/account/me/2fa/setup` | Begin 2FA setup (returns secret + QR) |
| `POST` | `/account/me/2fa/confirm` | Confirm 2FA, receive recovery codes |
| `DELETE` | `/account/me/2fa` | Disable 2FA |
| `GET` | `/account/me/sessions` | List active sessions |
| `DELETE` | `/account/me/sessions` | Revoke all other sessions |
| `DELETE` | `/account/me/sessions/:id` | Revoke a specific session |

`GET /health` is available for liveness checks.

---

## Security notes

A few decisions worth calling out, since they're the point of the project:

- **Sessions over JWTs.** Stateless tokens can't be revoked before they expire. Server-side sessions in Redis can — which is what makes logout, "sign out of all devices", and session listing actually meaningful.
- **Recovery codes are consumed atomically.** Consumption is a single `UPDATE ... SET recovery_codes = array_remove(...) WHERE id = $1 AND $2 = ANY(recovery_codes)`. Under concurrent requests with the same code, row-level locking guarantees exactly one succeeds — no read-modify-write race.
- **The pending-2FA state is isolated.** It lives in its own session field, not bolted onto an authorization role, so a half-authenticated session can never be mistaken for a full one.
- **Rate limiting keys on client IP**, because login endpoints are unauthenticated — there is no user identity to key on yet.
- **Passwords are validated strictly on the way in, loosely on the way back.** Registration enforces length bounds; login only checks presence, so tightening the password policy later never locks out existing users before their credentials are even checked.

---

## Intentionally out of scope

Warden is authentication, not a user-management platform. The following are deliberately absent — adding any of them on top is trivial, while removing them from a bloated starter is not:

- **No roles / RBAC.** Every operation acts on the caller's own account; a role field would be dead weight until there's a real admin surface to guard.
- **No profile fields** (name, avatar, etc.) — that's profile management, a separate concern.
- **Schema via init script** is a development convenience; a production deployment should manage the schema with migrations.

---

## License

MIT — see [LICENSE](LICENSE).
