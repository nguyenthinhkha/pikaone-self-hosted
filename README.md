# PikaOne Self-Hosted

Run PikaOne with Docker Compose using pre-built images from Docker Hub.

```bash
git clone https://github.com/nguyenthinhkha/pikaone-self-hosted.git
cd pikaone-self-hosted
cp .env.example .env
cp .env.db.example .env.db
cp .env.api.example .env.api
# Edit .env.db, .env.api — set JWT_SECRET, POSTGRES_PASSWORD, SEED_ADMIN_*, RESEND_*, OPENAI_API_KEY, WEB_BASE_URL, MASTER_KEY
docker compose -p pikaone --env-file .env up -d
```

## First boot

On startup the API container runs database migrations, then seeds reference data (assets and currency units), then creates the first administrator when `SEED_ADMIN_EMAIL` and `SEED_ADMIN_PASSWORD` are set in `.env.api`. Set both before the first `docker compose up`.

- Assets are upserted on every API start (safe to re-run).
- The administrator is created only once per email address.
- Change the admin password after first login.

For an existing deployment that was started without seed env vars:

```bash
docker compose -p pikaone exec app npm run seed:prod
```

Then restart the API container if it was already running.

## Application-level encryption

Sensitive fields (account names, balances, transaction amounts, notes, and similar) can be encrypted at rest with AES-256-GCM. Each user has a data-encryption key (DEK) wrapped by `MASTER_KEY`. The API decrypts data in memory before returning JSON to the web app.

**Back up `MASTER_KEY` securely.** If it is lost, encrypted data cannot be recovered.

### New deployment

1. Generate a key: `openssl rand -base64 32`
2. Add to `.env.api`:
   - `MASTER_KEY=<generated value>`
   - `ENCRYPTION_ENABLED=true` (optional; default `false` stores plaintext until you enable it)
3. Start the stack. New users receive a DEK on registration; no data migration is needed on an empty database.

### Enable encryption on an existing deployment

Run these steps after upgrading to an image that includes the encryption migration (`20260702143547_add_encryption_layer`):

1. Add `MASTER_KEY` to `.env.api` (generate with `openssl rand -base64 32`). Keep `ENCRYPTION_ENABLED=false` for now.
2. Pull and restart so migrations apply:
   ```bash
   docker compose -p pikaone pull
   docker compose -p pikaone --env-file .env up -d
   ```
3. Encrypt existing rows (requires `MASTER_KEY`; the script enables encryption for the run only):
   ```bash
   docker compose -p pikaone exec app node -r reflect-metadata dist/src/scripts/encrypt-existing-data.js
   ```
4. Set `ENCRYPTION_ENABLED=true` in `.env.api` and restart the API container:
   ```bash
   docker compose -p pikaone --env-file .env up -d app
   ```

Authenticated users can check rotation status and start DEK rotation via `GET/POST /api/users/me/encryption/*` (see Swagger).

## Prerequisites

- Docker Engine 24+ with Compose v2

## URLs

| Service | URL |
|---------|-----|
| Web | http://localhost:3000 |
| API | http://localhost:4000/api |
| Swagger | http://localhost:4000/doc |
| Grafana (optional) | http://localhost:3001 |

## Updating

```bash
docker compose -p pikaone pull
docker compose -p pikaone --env-file .env up -d
```

Pin a release: check out a git tag (e.g. `26.07.01`) and set `PIKAONE_IMAGE_TAG=26.07.01` in `.env`.

## Optional observability

Logs (Loki), traces (Tempo), and metrics (Prometheus) via Grafana:

```bash
cp observability/.env.example observability/.env
# Edit Grafana admin password before first boot
docker compose -p pikaone-observability \
  -f observability/docker-compose.yml \
  --env-file observability/.env up -d
```

Start the app stack first so the shared `pikaone` Docker network exists.

## Volumes

| Volume | Purpose |
|--------|---------|
| `pikaone_db_data` | PostgreSQL data |
| `pikaone_redis_data` | Redis persistence |
| `pikaone_uploads` | API file uploads |
| `pikaone_grafana_data`, etc. | Observability (if enabled) |

## Environment variables

### Compose (`.env`)

| Variable | Default | Description |
|----------|---------|-------------|
| `PIKAONE_IMAGE_TAG` | `latest` | Docker tag for `pikaone-api` and `pikaone-web` (`latest` or `YY.MM.DD`) |

### `nguyenthinhkha/pikaone-api` (`.env.api`)

| Variable | Required | Description |
|----------|----------|-------------|
| `PORT` | No | Listen port (default `4000`) |
| `DATABASE_URL` | Yes | Postgres URL; host must be `db` |
| `JWT_SECRET` | Yes | Token signing secret |
| `JWT_ACCESS_TOKEN_EXPIRES_IN` | No | Access token TTL (e.g. `7d`) |
| `JWT_REFRESH_TOKEN_EXPIRES_IN_DAYS` | No | Refresh token lifetime in days |
| `PASSWORD_RESET_TOKEN_EXPIRES_IN_HOURS` | No | Password reset link TTL |
| `RESEND_API_KEY` | Yes* | Resend email API key (*for email features) |
| `RESEND_FROM_EMAIL` | Yes* | Sender address for transactional email |
| `REDIS_HOST` / `REDIS_PORT` | Overridden | Compose sets `redis:6379` |
| `REDIS_PASSWORD` / `REDIS_DB` | No | Optional Redis auth/DB index |
| `WEB_BASE_URL` | Yes | Public web URL for email links |
| `SEED_ADMIN_EMAIL` | Yes* | First administrator email (*first boot) |
| `SEED_ADMIN_PASSWORD` | Yes* | Initial admin password (*first boot; change after login) |
| `OPENAI_API_KEY` | No | Required for OpenAI chat/vision |
| `OPENAI_CHAT_MODEL` / `OPENAI_VISION_MODEL` | No | Model names |
| `CHAT_LLM_PROVIDER` | No | `openai` or `ollama` |
| `OLLAMA_BASE_URL` / `OLLAMA_MODEL` | No | When using Ollama |
| `VNAPP_MOB_GOLD_API_KEY` | No | VN domestic gold prices |
| `MASTER_KEY` | Yes* | Base64-encoded 32-byte key for field encryption (*required when `ENCRYPTION_ENABLED=true`; generate with `openssl rand -base64 32`) |
| `ENCRYPTION_ENABLED` | No | `true` to encrypt sensitive fields at rest (default `false`) |
| `ENCRYPTION_DEK_CACHE_TTL_SECONDS` | No | Redis TTL for per-user DEK cache (default `1800`) |
| `LOG_LEVEL` / `LOG_PRETTY` | No | Use `LOG_PRETTY=false` for Loki JSON logs |
| `OTEL_SERVICE_NAME` | No | Default `pikaone-api` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | No | `http://otel-collector:4317` when observability is up |
| `OTEL_RESOURCE_ATTRIBUTES` | No | e.g. `deployment.environment=production` |
| `ASSET_PRICES_*` | No | Scheduler cron overrides |

### `nguyenthinhkha/pikaone-web`

| Variable | When | Description |
|----------|------|-------------|
| `NEXT_PUBLIC_API_BASE_URL` | Build-time | Baked as `/api` |
| `API_BASE_URL` | Build-time | Baked as `http://pikaone-api:4000` |
| `PORT` / `HOSTNAME` | Runtime | Set in image; no user config needed |

### `postgres:18.3` (`.env.db`)

| Variable | Required | Description |
|----------|----------|-------------|
| `POSTGRES_USER` | Yes | Must match `DATABASE_URL` user |
| `POSTGRES_PASSWORD` | Yes | Must match `DATABASE_URL` password |
| `POSTGRES_DB` | Yes | Database name |
| `POSTGRES_INITDB_ARGS` | No | Init encoding/locale |

### `redis:8.6.2`

No env file required. Persistence via `pikaone_redis_data` volume.

### Observability (`observability/.env`)

| Variable | Service | Description |
|----------|---------|-------------|
| `GRAFANA_ADMIN_USER` | grafana | Initial admin user (first boot only) |
| `GRAFANA_ADMIN_PASSWORD` | grafana | Initial admin password (first boot only) |

Loki, Promtail, Tempo, Prometheus, and OTEL Collector use mounted YAML configs.

---

> **Maintainers:** Source lives in the `pikaone` monorepo under `hosts/self-hosted/` and `hosts/observability/`. Do not edit this repository directly — run `scripts/publish-self-hosted-repo.sh` from the monorepo after changes.
