# Railway template wiring

Publish this as a Railway template named `manyfold` (not `template-manyfold`).

Railway does not deploy `docker-compose.yml` directly. Recreate its topology in
the template composer with services named exactly `app`, `postgres`, and
`redis`.

Only **`app`** should be public. Postgres and Redis stay private.

## Application services

Use `https://github.com/osbytes/template-manyfold` as the source for `app`.
Set the service root directory as shown; its local `railway.toml` sets the
Dockerfile and healthcheck.

### app

- Root directory: `/services/app`
- Public networking: enabled (this is the only URL users open)
- Recommended memory: at least 2 GB (geometric analysis / conversion jobs are heavy)
- Variables:

```text
PORT=3214
PUID=1000
PGID=1000
SECRET_KEY_BASE=${{secret(64, "0123456789abcdef")}}
REDIS_URL=redis://default:${{redis.REDIS_PASSWORD}}@${{redis.RAILWAY_PRIVATE_DOMAIN}}:6379/1
DATABASE_ADAPTER=postgresql
DATABASE_HOST=${{postgres.RAILWAY_PRIVATE_DOMAIN}}
DATABASE_PORT=5432
DATABASE_USER=${{postgres.POSTGRES_USER}}
DATABASE_PASSWORD=${{postgres.POSTGRES_PASSWORD}}
DATABASE_NAME=${{postgres.POSTGRES_DB}}
MULTIUSER=enabled
HTTPS_ONLY=enabled
PUBLIC_HOSTNAME=${{RAILWAY_PUBLIC_DOMAIN}}
WEB_CONCURRENCY=2
RAILS_MAX_THREADS=8
```

Keep the public domain target port at `3214` (image default; also matches
upstream docs). Railway injects `PORT` on some plans — if the platform overrides
it, leave `PORT` unset and point public networking at the injected value
instead; Puma listens on `PORT`.

Volume (one per service — Railway limit;
https://station.railway.com/feedback/multiple-volumes-per-service-7ce57788):

| Mount path | Purpose |
|------------|---------|
| `/models` | Model library filesystem (create a library pointed here in the UI) |

Upstream Compose also mounts `/usr/src/app/plugins`. On Railway leave that path
**ephemeral** (no second volume). Plugins installed via the UI will not survive
redeploys unless you later fold them under `/models` somehow.

Healthcheck path: `/health` (timeout 300s — first boot runs `db:prepare`).

Optional deployer variables (leave unset unless needed):

```text
FEDERATION=
OIDC_CLIENT_ID=
OIDC_CLIENT_SECRET=
OIDC_ISSUER=
OIDC_NAME=
SMTP_SERVER=
SMTP_PORT=
SMTP_USERNAME=
SMTP_PASSWORD=
SMTP_FROM_ADDRESS=
PUBLIC_PORT=
```

If enabling `FEDERATION=enabled`, keep the app at the domain root (no
`RAILS_RELATIVE_URL_ROOT`) and ensure `PUBLIC_HOSTNAME` / `HTTPS_ONLY` are set.

## Data services

### postgres

- Image: `postgres:15`
- Public networking: disabled
- Volume: mount at `/var/lib/postgresql/data`
- Variables:

```text
POSTGRES_DB=manyfold
POSTGRES_USER=manyfold
POSTGRES_PASSWORD=${{secret(32)}}
```

### redis

Prefer Railway's **Redis** template/plugin (Redis 8). It enables auth by default;
Manyfold will crash with `NOAUTH Authentication required` if `REDIS_URL` omits
the password.

If you wire Redis yourself instead of the plugin:

- Image: `redis:8`
- Public networking: disabled
- Start command (auth required on Railway private networking is fine, but keep
  the password in sync with `REDIS_URL` on `app`):

```text
redis-server --requirepass ${{REDIS_PASSWORD}}
```

- Variables on the `redis` service:

```text
REDIS_PASSWORD=${{secret(32)}}
```

- No volume required for a typical deploy (cache/job broker; jobs re-queue after
  restart). Add a volume at `/data` with a custom start command only if you need
  Redis persistence.

If the Redis service exposes a full URL variable (`REDIS_URL` /
`REDIS_PRIVATE_URL`), you may set `app`'s `REDIS_URL` to that reference instead
of constructing it. Keep `/1` only if you intentionally separate DB indexes;
database `0` is fine.

## Auth notes

- Open the **app** URL only.
- On first visit, set a secure administrator password when prompted.
- `MULTIUSER=enabled` is set by default for public Internet exposure. Single-user
  mode (unset / disabled) skips login — do not use that on a public domain.
- After creating your admin account, review signup/registration settings in the
  Manyfold admin UI if you do not want open registration.

## Deployment notes

- Keep all three services in one Railway environment so reference variables and
  private DNS resolve correctly.
- Prefer the standard `manyfold3d/manyfold` image (separate Postgres + Redis),
  not `manyfold-solo`, so Railway volumes and sizing stay explicit.
- First boot can take 1–2 minutes while migrations run; wait for `/health`.
- After deploy: open the public URL → set admin password → **Libraries → New**
  with path `/models` → upload a small STL/3MF to confirm storage + workers.
- Image pin: Docker Hub `manyfold3d/manyfold:0.146.0` (upstream release
  `v0.146.0`). GitHub Releases keep the `v` prefix; Docker Hub semver tags do not.

## Troubleshooting

| Symptom | Likely cause |
|---------|----------------|
| App crash-loops with `NOAUTH` / Redis auth | Railway Redis 8 requires a password; include `${{redis.REDIS_PASSWORD}}` (or use `${{redis.REDIS_URL}}`) |
| App crash-loops mentioning Redis host | `REDIS_URL` host wrong; must use `redis` private domain |
| App crash-loops on DB | Postgres not ready / wrong `DATABASE_*` refs |
| Blank or cookie errors over HTTPS | Missing `HTTPS_ONLY=enabled` or wrong `PUBLIC_HOSTNAME` |
| Cannot write models | Volume not mounted at `/models`, or PUID/PGID mismatch |
| Plugins missing after redeploy | Only `/models` is persisted; plugins path is ephemeral on Railway |
| Slow / OOM during analysis | Raise memory; lower `PERFORMANCE_WORKER_CONCURRENCY` if set high |
