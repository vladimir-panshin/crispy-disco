# Warden

**Session-based authentication service in Go** — TOTP 2FA, recovery codes, device-aware session management, and brute-force protection. Built on PostgreSQL and Redis.

![Go](https://img.shields.io/badge/Go-1.26-00ADD8?logo=go&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

Warden is an authentication backend — registration, login, two-factor authentication, and session lifecycle, and deliberately nothing else. The goal is a small, readable core that's easy to build on.

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
cp .env.example .env
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

---

## License

MIT — see [LICENSE](LICENSE).
