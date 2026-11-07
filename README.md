# IDENTERA

IDENTERA is a self-hostable authentication platform. Developers register apps, plug in their own OAuth credentials (Google, GitHub) or use email/password auth, and send users to a hosted login page. Backends then verify the issued JWTs with per-app signing keys published over JWKS.

## Features

- **Developer portal**: register, create apps, manage redirect URIs, rotate API secrets and signing keys, and view app users
- **Hosted login UI**: sign in, registration, password reset and email verification, using PKCE
- **OAuth providers**: Google and GitHub, using credentials each developer brings for their own app
- **Email auth**: register and log in with a password, verify email, and reset a password (emails sent through [Resend](https://resend.com))
- **Tokens**: access/refresh token pairs, refresh rotation, revocation, and a per-app JWKS endpoint
- **SDKs**: `identera-react` for the frontend and `identera-node` (Express middleware) for the backend
- **Rate limiting** on auth endpoints (Redis-backed)

## Architecture

```
                ┌──────────────────────┐
 Browser / SDK ─▶│  api-gateway  :8080  │  (Gin, HTTP/REST)
                └──────────┬───────────┘
                           │ gRPC
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
 developer-service   token-service       user-service
     :50051              :50052              :50053
        │                  │                  │
        └──────── PostgreSQL ─────── Redis ───┘
```

| Component | Path | Description |
|---|---|---|
| API Gateway | `services/api-gateway` | Public HTTP API, auth middleware, rate limiting |
| Developer Service | `services/developer-service` | Developers, apps, OAuth provider configs, signing keys |
| Token Service | `services/token-service` | JWT issuance, refresh, revocation, JWKS |
| User Service | `services/user-service` | End users, identities, sessions, email flows, OAuth |
| Protos | `proto/` | gRPC contracts (`developer`, `token`, `user`) |
| Migrations | `migrations/` | PostgreSQL schema (golang-migrate) |
| Dashboard | `frontend/dashboard` | Developer portal (Next.js, port 3000) |
| Login UI | `frontend/login-ui` | Hosted login pages (Next.js, port 3001) |
| SDKs | `sdk/identera-react`, `sdk/identera-node` | Client and server integrations |
| Demo | `demo/frontend-app`, `demo/backend-app` | Example app using both SDKs |

## Tech Stack

Go 1.25 · Gin · gRPC/Protobuf · PostgreSQL 16 · Redis 7 · Next.js · TypeScript · Tailwind CSS · Docker Compose

## Getting Started

### Prerequisites

- Go 1.25+
- Node.js 18+
- Docker and Docker Compose
- (Optional) `protoc` with `protoc-gen-go` / `protoc-gen-go-grpc` to regenerate protos
- (Optional) [golang-migrate](https://github.com/golang-migrate/migrate) to run migrations locally

### 1. Configure the environment

```bash
cp .env.example .env
```

Set at least `JWT_SECRET` and `ENCRYPTION_KEY` (32 bytes). Set `RESEND_API_KEY` and `EMAIL_FROM` to send real emails.

### 2. Run the backend with Docker

```bash
make docker-up      # postgres, redis, migrations, all services
```

The API gateway is now at `http://localhost:8080`.

### Or run the services locally

```bash
docker-compose up -d postgres redis
make migrate-up
make run-developer   # each in its own terminal
make run-token
make run-user
make run-gateway
```

### 3. Run the frontends

```bash
cd frontend/dashboard && npm install && npm run dev   # http://localhost:3000
cd frontend/login-ui  && npm install && npm run dev   # http://localhost:3001
```

## Makefile Targets

| Target | Description |
|---|---|
| `make docker-up` / `docker-down` / `docker-clean` | Start, stop, or wipe the Docker stack |
| `make run-gateway` / `run-developer` / `run-user` / `run-token` | Run one service locally |
| `make proto` | Regenerate Go code from `.proto` files |
| `make migrate-up` / `migrate-down` / `migrate-create` | Manage database migrations |
| `make tidy` | `go mod tidy` |

## API Overview

**Developer portal** (`/api/v1/developers`, JWT required except register/login)

- `POST /register`, `POST /login`, `POST /logout`
- `GET|PATCH /profile`
- `POST|GET /apps`, `GET|PATCH|DELETE /apps/:id`
- `GET /apps/:id/users`
- `POST /apps/:id/rotate-secret`, `POST /apps/:id/rotate-keys`, `GET /apps/:id/keys`
- `POST|GET /apps/:id/providers`, `PATCH|DELETE /apps/:id/providers/:provider`

**OAuth**

- `GET /oauth/authorize?app_id=…&provider=…&redirect_uri=…`
- `GET /oauth/callback/:provider`
- `POST /oauth/refresh`, `POST /oauth/revoke`

**Email auth**

- `POST /auth/register`, `POST /auth/login`
- `POST /auth/forgot-password`, `POST /auth/reset-password`, `POST /auth/verify-email`

**Public / verification**

- `GET /api/v1/public/apps/:app_id`
- `GET /api/v1/apps/:app_id/jwks`
- `POST /api/v1/verify` (API key required)
- `GET /api/v1/userinfo`

> When you configure Google or GitHub, add the **IDENTERA callback** (`http://localhost:8080/oauth/callback/{provider}`) to the IdP's allowlist, not your app's redirect URI. Register your app's own redirect URI in the IDENTERA dashboard.

## SDK Usage

### React

```tsx
import { AuthCallbackHandler, AuthGuard, IdenteraProvider, useAuth } from 'identera-react';

function LoginButton() {
  const { login } = useAuth();
  return <button onClick={() => login()}>Sign in</button>;
}

export default function App() {
  return (
    <IdenteraProvider
      appId={process.env.NEXT_PUBLIC_AUTH_PUBLISHABLE_KEY!}
      authUrl={process.env.NEXT_PUBLIC_AUTH_URL!}
      redirectUri={process.env.NEXT_PUBLIC_AUTH_REDIRECT_URI!}
    >
      <AuthCallbackHandler />
      <AuthGuard fallback={<LoginButton />}>
        <div>Protected app content</div>
      </AuthGuard>
    </IdenteraProvider>
  );
}
```

### Node.js (Express)

```js
import { requireAuth } from 'identera-node';

app.get('/protected', requireAuth({
  appId: process.env.AUTH_APP_ID,
  apiUrl: process.env.AUTH_API_URL,
  issuer: process.env.AUTH_ISSUER,
}), (req, res) => {
  res.json({ user: req.auth });
});
```

## Demo

`demo/` holds a full example app: a Next.js frontend (port 3000) and an Express backend (port 4000).

```bash
cd demo/backend-app  && cp .env.example .env && npm install && npm run dev
cd demo/frontend-app && cp .env.example .env && npm install && npm run dev
```

Create an app in the dashboard, then copy its publishable key into both `.env` files.
